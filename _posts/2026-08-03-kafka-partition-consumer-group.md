---
title: "카프카 파티션과 컨슈머 그룹 실전 — 병렬 처리, 리밸런싱, 순서 보장, 장애 복구"
description: 파티션 개수가 왜 병렬 처리량을 결정하는지, 컨슈머 그룹이 파티션을 어떻게 자동 배분하고 리밸런싱하는지, 메시지 순서는 언제 보장되고 언제 안 되는지, 컨슈머가 죽었다 살아나면 어떻게 되는지를 직접 확인해본 실습 노트입니다.
author: yoonxjoong
date: 2026-08-03 11:00:00 +0900
categories:
  - Backend
tags:
  - Kafka
  - Docker
---

## 준비: 단일 브로커

이번엔 클러스터 대신 단일 브로커로 파티션/컨슈머 그룹 동작 자체에 집중했습니다.

```bash
docker network create kafka-net

docker run -d --name zookeeper \
  --network kafka-net \
  -p 2181:2181 \
  -e ZOOKEEPER_CLIENT_PORT=2181 \
  -e ZOOKEEPER_TICK_TIME=2000 \
  confluentinc/cp-zookeeper:7.5.0

docker run -d --name kafka-server \
  --network kafka-net \
  -p 9092:9092 \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  confluentinc/cp-kafka:7.5.0
```

카프카는 메시지를 메모리에만 두는 게 아니라 물리적인 파일로 저장합니다. 파티션을 여러 개로 나누면 그만큼 파일도 나뉘어 저장되는데, 이게 병렬 처리의 물리적 기반이 됩니다.

## 병렬 처리의 핵심 — 컨슈머는 파티션 하나에만 붙는다

**한 컨슈머는 한 파티션에만 붙을 수 있습니다.** 그래서:

- 파티션이 1개면, 컨슈머를 10대 준비해도 1대만 일하고 9대는 놉니다.
- 파티션이 3개면, 컨슈머 3대가 동시에 붙어서 처리량이 3배가 됩니다.

파티션 3개짜리 `order-topic`을 만들고, 각 파티션에 컨슈머를 직접 붙여서 확인했습니다.

```bash
# 파티션 3개인 order-topic 생성
docker exec -it kafka-server kafka-topics \
  --create --topic order-topic \
  --partitions 3 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092

# 파티션을 지정해서 컨슈머 3개 실행
docker exec -it kafka-server kafka-console-consumer --topic order-topic --bootstrap-server localhost:9092 --partition 0
docker exec -it kafka-server kafka-console-consumer --topic order-topic --bootstrap-server localhost:9092 --partition 1
docker exec -it kafka-server kafka-console-consumer --topic order-topic --bootstrap-server localhost:9092 --partition 2

# 프로듀서 실행
docker exec -it kafka-server kafka-console-producer --topic order-topic --bootstrap-server localhost:9092

# 키를 부여해서 파티션을 분리
docker exec -it kafka-server kafka-console-producer --topic order-topic --bootstrap-server localhost:9092 \
  --property "parse.key=true" --property "key.separator=:"
```

## 컨슈머 그룹 — 자동 배분과 장애 복구

파티션에 컨슈머를 직접 지정하는 대신, 여러 컨슈머를 하나의 **컨슈머 그룹**으로 묶으면 두 가지를 카프카가 대신 해줍니다.

- **자동 배분**: 그룹 이름만 같으면, 브로커가 파티션 0/1/2를 컨슈머들에게 알아서 나눠줍니다.
- **장애 복구**: 컨슈머 하나가 죽으면, 그 컨슈머가 맡던 파티션을 남은 컨슈머들에게 다시 배분(리밸런싱)합니다.

같은 그룹(`my-order-group`)으로 컨슈머 3개를 띄우고, 프로듀서로 메시지를 보내면서 상태를 확인했습니다.

```bash
# 컨슈머 3개, 전부 같은 그룹으로 실행
docker exec -it kafka-server kafka-console-consumer --topic order-topic --bootstrap-server localhost:9092 --group my-order-group
# (터미널 3개에서 각각 동일하게 실행)

# 프로듀서 실행
docker exec -it kafka-server kafka-console-producer --topic order-topic --bootstrap-server localhost:9092

# 컨슈머 그룹 상태 확인
docker exec -it kafka-server kafka-consumer-groups --bootstrap-server localhost:9092 --group my-order-group --describe
```

`--describe` 결과의 컬럼들은 이렇게 읽으면 됩니다.

- **PARTITION**: 토픽 내 파티션 번호 (0, 1, 2)
- **CURRENT-OFFSET**: 해당 컨슈머가 지금까지 읽은 마지막 메시지 번호
- **LOG-END-OFFSET**: 브로커에 쌓여 있는 마지막 메시지 번호
- **LAG**: 아직 못 읽은 메시지 수 (`LOG-END-OFFSET - CURRENT-OFFSET`)
- **CONSUMER-ID**: 어떤 컨슈머가 이 파티션을 맡고 있는지 나타내는 고유 식별자

컨슈머 3개, 파티션 3개라 정확히 1:1로 배분됐습니다.

## 컨슈머가 파티션보다 많으면? — 잉여 컨슈머

여기에 컨슈머를 1개 더 추가(4번째)하고 `--members` 옵션으로 확인해봤습니다.

```bash
docker exec -it kafka-server kafka-console-consumer --topic order-topic --bootstrap-server localhost:9092 --group my-order-group
```

파티션은 3개인데 컨슈머는 4개라, 4번째 컨슈머는 **PARTITIONS가 0**으로 나옵니다 — 맡을 파티션이 없어서 그냥 대기(예비) 상태로 남는 겁니다. 컨슈머를 늘린다고 무조건 처리량이 느는 게 아니라, **파티션 수가 병렬 처리의 상한선**이라는 걸 직접 확인한 셈입니다.

## 파티션을 늘리면 즉시 반영될까? — 리밸런싱은 트리거가 필요하다

컨슈머 4개(그중 1개는 예비)인 상태에서 파티션을 3개에서 4개로 늘려봤습니다.

```bash
docker exec -it kafka-server kafka-topics --alter --topic order-topic --partitions 4 --bootstrap-server localhost:9092
```

파티션을 늘린 직후 바로 컨슈머 정보를 확인하면 **그대로**입니다 — 파티션이 하나 늘었다고 컨슈머가 즉시 새 파티션을 받아가는 게 아닙니다. 리밸런싱은 트리거가 발동돼야 시작되고, 컨슈머들이 "새 파티션이 생겼다"는 걸 인지하는 과정이 필요합니다. 이후 실제로 리밸런싱이 일어나면 0~3번까지 파티션이 각 컨슈머에 할당되는 걸 확인했고, `--describe --members`로 보면 예비였던 4번째 컨슈머까지 포함해서 **PARTITIONS가 전부 1**로, 4개 파티션이 4개 컨슈머에 고르게 재분배된 걸 확인했습니다.

## 메시지 순서는 언제 보장되나

주문 → 결제 → 배송처럼 순서가 중요한 데이터를 다룰 때 헷갈리기 쉬운 지점입니다.

- **같은 파티션 안에서는 순서가 보장됩니다.** 주문과 결제가 둘 다 파티션 0번에 들어가면, 들어간 순서 그대로 처리됩니다.
- **다른 파티션에 걸치면 순서가 보장되지 않습니다.** 주문이 파티션 0번, 결제가 파티션 1번에 들어가면, 컨슈머 처리 속도에 따라 결제가 먼저 처리될 수도 있습니다.

## 키로 순서 보장하기

**같은 키를 가진 메시지는 항상 같은 파티션으로 전송됩니다.** 그래서 특정 주문에 관련된 데이터를 전부 같은 키(예: 주문번호)로 보내면, 그 주문에 한해서는 순서가 보장됩니다.

```bash
docker exec -it kafka-server kafka-console-producer --topic order-topic --bootstrap-server localhost:9092 \
  --property "parse.key=true" --property "key.separator=:"
```

```
> order123:step1_create
> order123:step2_pay
> order123:step3_ship
> order555:step1_create
> order123:step4_shipping
```

`order123`으로 보낸 메시지들은 전부 같은 파티션에, 같이 연결된 같은 컨슈머로 순서대로 전송되는 걸 확인했습니다. `order555`는 (키가 다르니) 다른 파티션으로 갈 수 있지만, `order123`끼리는 순서가 섞이지 않습니다.

## 컨슈머가 처리 중에 죽으면?

컨슈머 하나를 강제로 종료시키고, 프로듀서로 메시지를 10~20개 정도 더 발행한 뒤 상태를 확인했습니다. `--describe`로 보면 **LAG이 20** 정도로 나오는데, 이건 "아직 못 읽은 메시지가 20개 있다"는 뜻입니다.

이후 죽었던 컨슈머를 다시 그룹에 합류시키면, **마지막으로 커밋된 오프셋부터 이어서 읽습니다** — 처음부터 다시 읽지도, 죽어있던 동안 쌓인 메시지를 건너뛰지도 않습니다. 컨슈머가 메시지를 처리할 때마다 오프셋을 커밋해두기 때문에, 죽었다 살아나도 "어디까지 읽었는지"를 잃어버리지 않는 겁니다.

## 참고 자료

- [Confluent Platform Docker 이미지](https://hub.docker.com/r/confluentinc/cp-kafka)
