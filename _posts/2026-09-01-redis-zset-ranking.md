---
title: "정렬 테이블 랭킹을 Redis ZSET으로 바꾸면 무엇이 달라지는가"
description: JSONB 정렬 테이블로 상품 랭킹을 구현했던 이전 글에는 전체 배열 재작성, 단일 row 낙관적 락으로 인한 충돌, 특정 상품 순위 조회 시 O(n) 스캔이라는 문제가 남아 있었습니다. Redis Sorted Set(ZSET)이 이 문제들을 구조적으로 어떻게 다르게 처리하는지 정리한 설계 비교 글입니다.
author: yoonxjoong
date: 2026-09-01 10:00:00 +0900
categories:
  - Backend
tags:
  - Redis
  - Architecture
  - Ranking
---

> [이전 글](/posts/ranking-architecture/)에서는 순번 컬럼 방식의 문제(연쇄 UPDATE, 데드락)를 JSONB 정렬 테이블 + 낙관적 락 + 캐싱으로 개선했습니다. 이 글은 그 JSONB 방식에 남아 있던 문제를 짚고, Redis Sorted Set(ZSET)으로 바꾸면 같은 요구사항을 어떤 방식으로 다르게 처리하는지 정리한 설계 비교입니다. 실제로 부하를 걸어 측정한 벤치마크는 아니고, 두 자료구조의 동작 방식 차이를 근거로 비교한 내용입니다.

## JSONB 정렬 테이블 방식에 남아 있던 문제

이전 글의 `ranking_order` 테이블은 카테고리 하나당 상품 ID 배열을 JSONB 컬럼 하나에 통째로 저장합니다.

```sql
CREATE TABLE ranking_order (
    id BIGINT PRIMARY KEY,
    category_id BIGINT,
    product_ids JSONB,
    version BIGINT,
    UNIQUE (category_id)
);
```

이 구조에서 순위를 하나만 바꿔도 실제로 일어나는 일은 다음과 같습니다.

**전체 배열 재작성.** `updateRanking`은 상품 하나의 위치만 바뀌어도 `product_ids` 배열 전체를 새로 만들어서 그 row 하나를 통째로 다시 씁니다. 배열에 상품이 5개든 500개든, 쓰기 단위는 항상 "카테고리 하나의 전체 순위"입니다.

**단일 row 낙관적 락으로 인한 충돌.** `@Version`이 걸려 있는 대상이 카테고리 row 하나이기 때문에, 같은 카테고리 안에서 서로 다른 상품을 편집하는 두 관리자도 같은 row를 두고 경쟁합니다. 관리자 A가 1번 상품을 2등으로, 관리자 B가 3번 상품을 4등으로 거의 동시에 옮기면, 두 변경이 논리적으로는 겹치지 않는데도 버전 충돌로 한쪽은 재시도해야 합니다.

**특정 상품의 현재 순위 조회가 O(n).** `getRankedProducts`는 `product_ids` 리스트를 순회하면서 `products` 리스트를 매번 다시 필터링합니다.

```java
return productIds.stream()
    .map(id -> products.stream()
        .filter(p -> p.getId().equals(id))
        .findFirst()
        .orElse(null))
    .filter(Objects::nonNull)
    .collect(Collectors.toList());
```

바깥 `map`과 안쪽 `filter`가 중첩돼 있어서 실제로는 O(n²)이고, "이 상품이 지금 몇 등인가"만 알고 싶어도 배열 전체를 순회해야 합니다.

**이력 테이블의 저장 공간 증폭.** `ranking_order_history`는 변경이 있을 때마다 그 시점의 `product_ids` 배열 전체를 복제해서 저장합니다. 바뀐 건 상품 하나의 위치인데, 저장되는 이력 row는 카테고리 전체 순위의 스냅샷입니다.

## Redis Sorted Set의 구조

ZSET은 `(member, score)` 쌍의 집합이고, score 기준으로 항상 정렬된 상태를 유지합니다. Redis 내부적으로는 skip list와 hash table을 함께 유지하는데, skip list가 순서 기반 연산(순위 조회, 범위 조회)을, hash table이 멤버 기반 연산(특정 멤버의 score 조회)을 각각 담당합니다. 그래서 삽입·삭제·순위 조회는 O(log N), 특정 멤버의 score 조회는 O(1)입니다.

랭킹 도메인에 매핑하면 member는 상품 ID, score는 순위를 나타내는 값이 됩니다.

```java
private final RedisTemplate<String, String> redisTemplate;

public void upsertRank(Long categoryId, Long productId, double score) {
    String key = "ranking:category:" + categoryId;
    redisTemplate.opsForZSet().add(key, productId.toString(), score);
}

public List<Long> getTopN(Long categoryId, int n) {
    String key = "ranking:category:" + categoryId;
    Set<String> productIds = redisTemplate.opsForZSet()
        .reverseRange(key, 0, n - 1);
    return productIds.stream().map(Long::valueOf).toList();
}

public Long getRank(Long categoryId, Long productId) {
    String key = "ranking:category:" + categoryId;
    return redisTemplate.opsForZSet().reverseRank(key, productId.toString());
}
```

## 무엇이 구조적으로 달라지는가

**쓰기 단위가 멤버 하나로 줄어듭니다.** `ZADD`는 지정한 멤버의 score만 바꿉니다. 카테고리에 상품이 몇 개 있든 상관없이, 순위 하나를 바꾸는 연산의 크기는 그 상품 하나에만 비례합니다. JSONB 방식처럼 배열 전체를 다시 쓰는 과정이 없습니다.

**서로 다른 상품에 대한 동시 변경이 충돌하지 않습니다.** 낙관적 락은 "이 row를 읽은 시점과 쓰는 시점 사이에 다른 트랜잭션이 끼어들지 않았는가"를 카테고리 row 단위로 검사합니다. ZSET에는 그 단위 자체가 없습니다. 관리자 A가 상품 1의 score를 바꾸는 `ZADD`와 관리자 B가 상품 3의 score를 바꾸는 `ZADD`는 서로 다른 멤버를 대상으로 하는 별개의 원자적 연산이라, 재시도할 버전 충돌이 애초에 발생하지 않습니다. 단, 두 상품이 서로 정확히 자리를 맞바꾸는 경우(A를 3등, B를 1등으로 동시에)는 여전히 두 번의 `ZADD`가 순차적으로 나가는 것이므로, 그 사이 시점의 일관성까지 보장하려면 `MULTI`나 Lua 스크립트로 묶어야 합니다. 개별 명령의 원자성과 여러 명령 묶음의 원자성은 다른 문제입니다.

**특정 상품의 순위 조회가 O(log N)으로 줄어듭니다.** `ZREVRANK`는 hash table에서 멤버를 찾아 skip list 상의 위치를 계산하는 연산이라 O(log N)입니다. JSONB 방식의 `getRankedProducts`처럼 배열을 순회하며 필터링하는 코드 자체가 필요 없어집니다.

**범위 조회가 정렬된 상태로 바로 나옵니다.** `ZREVRANGE`는 요청한 구간을 이미 정렬된 순서로 반환합니다. JSONB 배열을 읽어와서 애플리케이션 레벨에서 순서를 다시 맞추는 코드가 필요 없습니다.

**중간 삽입이 다른 멤버의 재정렬 없이 가능합니다.** score는 정수가 아니어도 되므로, 두 상품 사이에 새 상품을 넣고 싶으면 그 두 score의 중간값을 주면 됩니다.

```java
public void insertBetween(Long categoryId, Long productId, double before, double after) {
    upsertRank(categoryId, productId, (before + after) / 2);
}
```

순번 컬럼 방식이 겪었던 "삽입 위치 이후 전체를 한 칸씩 미는 연쇄 UPDATE"가 구조적으로 발생하지 않습니다. 다만 이 방식을 반복하면 score 간격이 계속 좁아지다가 부동소수점 정밀도 한계에 부딪힐 수 있어서, 실무에서는 점수를 처음부터 큰 간격(예: 1000 단위)으로 배치해두는 것이 일반적입니다.

## 이 방식이 못 하는 것

**변경 이력이 자동으로 남지 않습니다.** JSONB 방식의 `ranking_order_history`는 별도 코드 없이 매 변경마다 스냅샷이 쌓였습니다. ZSET은 현재 상태만 들고 있기 때문에, "누가 언제 무엇을 바꿨는가"가 필요하다면 `ZADD`를 호출하는 지점에서 별도로 이력을 기록하는 코드를 붙여야 합니다. 이건 ZSET이 해결하는 문제가 아니라 ZSET으로 옮기면서 새로 떠안는 문제입니다.

**내구성 모델이 다릅니다.** Postgres는 커밋된 트랜잭션의 내구성을 기본으로 보장하지만, Redis는 RDB 스냅샷이나 AOF 설정에 따라 장애 시점과 마지막으로 디스크에 반영된 시점 사이의 변경이 유실될 수 있습니다. 랭킹 데이터를 Redis에만 두고 원본으로 취급할지, 아니면 Postgres를 원본으로 유지하고 ZSET은 조회 최적화를 위한 파생 데이터로 두고 필요시 재구성 가능하게 만들지는 별도로 결정해야 하는 문제입니다.

**상품 상세 정보는 여전히 다른 저장소가 필요합니다.** ZSET에는 상품 ID와 score만 있고, 이름·가격 같은 상품 정보는 없습니다. 순위 조회 결과로 얻은 ID 목록을 가지고 상품 저장소에 다시 조회하는 과정은 JSONB 방식과 동일하게 남아 있습니다. ZSET이 대체하는 건 "순서를 어떻게 저장하고 조회하는가"이지, 상품 데이터 자체의 저장소가 아닙니다.

## 참고 자료

- [Redis Sorted Sets](https://redis.io/docs/latest/develop/data-types/sorted-sets/) — 공식 문서
- [Redis ZADD](https://redis.io/docs/latest/commands/zadd/), [ZRANGE](https://redis.io/docs/latest/commands/zrange/), [ZRANK](https://redis.io/docs/latest/commands/zrank/) — 명령어 레퍼런스
- [Spring Data Redis Reference — Redis Repositories](https://docs.spring.io/spring-data/redis/reference/redis.html)
- [효율적인 상품 랭킹 시스템 설계](/posts/ranking-architecture/) — 이전 글, JSONB 정렬 테이블 방식
