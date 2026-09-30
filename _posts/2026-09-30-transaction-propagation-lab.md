---
title: "Spring 트랜잭션 전파 실습 검증 — REQUIRED, REQUIRES_NEW, NESTED, self-invocation"
description: "@Transactional의 전파 옵션을 Spring Boot와 PostgreSQL로 구성한 실습 프로젝트에서 직접 검증한 기록입니다. REQUIRED와 REQUIRES_NEW는 pg_backend_pid()로 물리 커넥션 단위까지 확인했고, NESTED는 예상과 달리 JPA 환경에서는 설정만으로 활성화할 수 없다는 사실을 바이트코드 분석으로 확인했습니다. self-invocation은 예외 없이 트랜잭션 보호만 사라진다는 것도 실측으로 확인했습니다."
author: yoonxjoong
date: 2026-09-30 09:00:00 +0900
categories:
  - Backend
tags:
  - Spring
  - Transaction
  - JPA
mermaid: true
---

[스프링 트랜잭션 개념 정리 글](/posts/spring-transaction-propagation-isolation/)에서 전파(Propagation)
7가지를 정리하면서, 아직 직접 재현해서 확인한 내용은 없다고 밝힌 바 있습니다. 이번에는 Spring Boot와
PostgreSQL로 실습 프로젝트를 구성해 직접 재현했으며, 예상과 다른 결과 두 가지를 확인했습니다.

실습 코드 전체는 [transaction-propagation-lab](https://github.com/yoonxjoong/transaction-propagation-lab)에서
확인할 수 있습니다.

## 검증 방법

"REQUIRED는 참여하고 REQUIRES_NEW는 새로 시작한다"는 설명은 로그만으로는 쉽게 납득되지만, 이것이 실제로
물리적인 DB 커넥션 수준에서 일어나는 일인지는 별도 확인이 필요합니다. 이를 위해 서비스 메서드마다
`pg_backend_pid()`를 기록해, 부모와 자식이 같은 커넥션을 사용하는지를 값으로 비교했습니다.

```java
@Component
public class ConnectionIdProbe {
    @PersistenceContext
    private EntityManager entityManager;

    public Integer currentBackendPid() {
        Object result = entityManager.createNativeQuery("select pg_backend_pid()").getSingleResult();
        return ((Number) result).intValue();
    }
}
```

## REQUIRED — 부모/자식이 물리적으로 같은 커넥션

```java
@Service
public class RequiredOrderService {

    @Transactional // REQUIRED (기본값)
    public void placeOrder(TxTrace trace, boolean failInInventory) {
        trace.record("outer-before", probe.currentBackendPid());
        orderRepository.save(new Order("required-demo"));
        inventoryService.decreaseStock(trace, failInInventory);
        trace.record("outer-after", probe.currentBackendPid());
    }
}

@Service
public class InventoryService {

    @Transactional // REQUIRED (기본값)
    public void decreaseStock(TxTrace trace, boolean fail) {
        trace.record("inner-required", probe.currentBackendPid());
        if (fail) {
            throw new IllegalStateException("재고 부족");
        }
    }
}
```

`decreaseStock`에서 예외가 발생하면 `placeOrder`의 `Order` insert까지 전부 롤백됩니다.
`outer-before`, `inner-required`, `outer-after` 세 시점의 pid는 모두 동일했습니다 — REQUIRED가 같은
물리 커넥션과 트랜잭션을 공유한다는 설명이 실제 동작과 일치함을 확인했습니다.

## REQUIRES_NEW — 부모가 롤백돼도 자식은 이미 커밋된 채로 남음

```java
@Service
public class RequiresNewOrderService {

    @Transactional // REQUIRED
    public void placeOrder(TxTrace trace, boolean failAfterAudit) {
        trace.record("outer-before", probe.currentBackendPid());
        orderRepository.save(new Order("requires-new-demo"));
        auditLogService.record(trace, "placeOrder attempted"); // REQUIRES_NEW
        trace.record("outer-after", probe.currentBackendPid());
        if (failAfterAudit) {
            throw new IllegalStateException("주문 처리 중 실패");
        }
    }
}

@Service
public class AuditLogService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(TxTrace trace, String message) {
        trace.record("inner-requires-new", probe.currentBackendPid());
        auditLogRepository.save(new AuditLog(message));
    }
}
```

```mermaid
sequenceDiagram
    participant Test
    participant TxA as placeOrder (Tx-A, pid=A)
    participant TxB as record (Tx-B, REQUIRES_NEW, pid=B)
    participant DB

    Test->>TxA: placeOrder(failAfterAudit=true)
    TxA->>DB: BEGIN (pid A)
    TxA->>TxB: record() 호출
    TxB->>DB: Tx-A suspend, BEGIN Tx-B (pid B, A와 다름)
    TxB->>DB: COMMIT Tx-B — AuditLog 즉시 확정
    TxB->>TxA: Tx-A resume (pid A로 복귀)
    TxA->>DB: ROLLBACK Tx-A — Order는 사라짐
    Note over DB: AuditLog는 이미 커밋되어 그대로 남아있음
```

`placeOrder(failAfterAudit=true)`를 호출하면 `Order`는 롤백되지만 `AuditLog`는 그대로 남습니다.
`outer-before`와 `inner-requires-new`의 pid는 서로 달랐고(별도 물리 커넥션), 자식 호출이 끝난 뒤
`outer-after`는 다시 `outer-before`와 같은 pid로 돌아왔습니다. 기존 트랜잭션을 suspend하고 새로
시작한다는 설명이 실제 커넥션 수준에서도 그대로 일어난다는 것을 확인했습니다.

## NESTED — 설정만으로는 활성화할 수 없다

개념 정리 글에서는 JDBC savepoint를 지원해야 동작하며 JPA 환경에서는 제약이 있다고 정리했습니다.
직접 검증하기 전에는 이를 설정으로 해결 가능한 제약으로 예상했지만, 실제로는 훨씬 근본적인
제약이었습니다.

가장 먼저 시도한 방법은 다음과 같습니다.

```java
@Bean
public PlatformTransactionManager nestedCapableTransactionManager(EntityManagerFactory emf) {
    JpaTransactionManager tm = new JpaTransactionManager(emf);
    tm.setNestedTransactionAllowed(true); // 기본값 false를 true로
    return tm;
}
```

그러나 이 매니저로 `@Transactional(propagation = Propagation.NESTED)`를 실행해도 다음 예외가 그대로
발생합니다.

```
NestedTransactionNotSupportedException: JpaDialect does not support savepoints
- check your JPA provider's capabilities
```

원인은 spring-orm 6.1.13의 클래스 파일을 직접 분석해 확인했습니다. `JpaTransactionManager`가
savepoint를 생성하려면, 트랜잭션 시작 시점에 `JpaDialect.beginTransaction()`이 반환하는 객체가
스프링의 `SavepointManager` 인터페이스를 구현하고 있어야 합니다. 그런데
`HibernateJpaDialect.beginTransaction()`은 `HibernateJpaDialect$SessionTransactionData`라는 객체를
반환하며, 이 클래스는 `SavepointManager`를 구현하지 않습니다. 즉 **`nestedTransactionAllowed` 플래그는
필요조건일 뿐 충분조건이 아니며, Hibernate 연동에는 애초에 savepoint 매니저를 생성하는 경로 자체가
존재하지 않습니다.** 설정 변경만으로 해결할 문제가 아니라, JPA에서 NESTED를 실제로 사용하려면
Hibernate 세션에서 JDBC 커넥션을 직접 꺼내 `SavepointManager`를 구현하는 커스텀 `JpaDialect`가
필요합니다.

이에 따라 NESTED가 실제로 동작하는 경로를 확인하기 위해 JPA 대신 순수 JDBC
(`DataSourceTransactionManager` + `JdbcTemplate`)로 전환했습니다.

```java
@Bean
public PlatformTransactionManager nestedCapableTransactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

```java
@Transactional(transactionManager = "nestedCapableTransactionManager", propagation = Propagation.REQUIRED)
public void runWithCapableManager(boolean failFinalCommit) {
    jdbcTemplate.update("insert into orders(product_name) values (?)", "nested-outer-capable");

    try {
        nestedItemService.saveItemWithCapableManager("nested-bad-item", true);
    } catch (RuntimeException ignored) {
        // savepoint까지만 롤백되고 배치는 계속 진행
    }

    nestedItemService.saveItemWithCapableManager("nested-good-item", false);

    if (failFinalCommit) {
        throw new IllegalStateException("배치 마무리 단계에서 실패 - 부모 전체 롤백");
    }
}
```

```java
@Transactional(transactionManager = "nestedCapableTransactionManager", propagation = Propagation.NESTED)
public void saveItemWithCapableManager(String name, boolean fail) {
    jdbcTemplate.update("insert into orders(product_name) values (?)", name);
    if (fail) {
        throw new IllegalStateException("아이템 저장 실패: " + name);
    }
}
```

이 구성으로 두 가지를 확인했습니다.

1. `nested-bad-item`은 savepoint까지만 롤백되고, `nested-good-item`과 배치 전체(`runWithCapableManager`)는
   정상 커밋됩니다.
2. 같은 배치에서 `failFinalCommit=true`로 마무리 단계를 한 번 더 실패시키면, 이미 savepoint를 통과해
   커밋된 것처럼 보였던 `nested-good-item`도 함께 사라집니다.

두 번째 결과가 핵심입니다. REQUIRES_NEW로 처리한 `AuditLog`는 부모가 이후 롤백되어도 살아남았지만,
NESTED로 처리한 `nested-good-item`은 부모가 최종적으로 롤백되는 순간 함께 사라집니다. **NESTED는
물리적으로 부모와 동일한 트랜잭션 안에서 savepoint만 생성하는 반면, REQUIRES_NEW는 별도의 물리
트랜잭션이라는 차이가 최종 결과에 그대로 드러납니다.**

## self-invocation — 예외 없이 조용히 트랜잭션 보호만 사라짐

같은 클래스 안에서 `this.method()`로 호출하면 프록시를 거치지 않아 `@Transactional`이 무시된다는 것은
잘 알려진 내용입니다. 이번 검증에서는 이를 값으로 증명하고, 한 걸음 더 나아가 그 상태에서 실제로 저장을
시도하면 어떤 일이 벌어지는지 확인했습니다.

```java
@Service
public class SelfInvocationDemoService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public boolean isActualTransactionActive() {
        return TransactionSynchronizationManager.isActualTransactionActive();
    }

    public boolean callTransactionalMethodViaSelfInvocation() {
        return this.isActualTransactionActive(); // 프록시를 거치지 않음
    }

    public void saveAuditViaSelfInvocation() {
        this.saveAuditRequiresNew();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditRequiresNew() {
        auditLogRepository.save(new AuditLog("self-invocation"));
    }
}
```

- 테스트에서 `selfInvocationDemoService.isActualTransactionActive()`를 직접(프록시를 거쳐) 호출하면
  `true`.
- 같은 빈 안에서 `this.isActualTransactionActive()`로 호출하면 `false` — `@Transactional(REQUIRES_NEW)`가
  명시돼 있어도 트랜잭션이 전혀 시작되지 않습니다.
- 그 상태로 `saveAuditViaSelfInvocation()`을 호출하면 `TransactionRequiredException`과 같은 예외가
  발생할 것으로 예상했으나, 실제로는 **예외 없이 저장이 완료됩니다.** Hibernate는 활성 트랜잭션이
  없으면 사실상 autocommit과 같은 방식으로 세션을 즉시 반영하기 때문입니다.

self-invocation이 위험한 이유는 여기에 있습니다. 트랜잭션 누락이 즉시 에러로 드러난다면 배포 전에
걸러지겠지만, 실제로는 평소에는 정상적으로 동작하는 것처럼 보이다가 롤백이 필요한 예외 상황에서만
문제가 드러납니다.

## 롤백 규칙 — checked exception은 기본값으로 롤백 안 됨

이 부분은 개념 정리 글의 설명과 일치했습니다. `@Transactional` 기본 설정에서는 checked
exception(`IOException`)을 던져도 `Order`가 그대로 커밋되며, `rollbackFor = Exception.class`를
명시해야 롤백됩니다. unchecked exception은 기본 설정으로도 롤백됩니다.

## 정리

| 항목 | 예상 | 실제 확인 결과 |
| --- | --- | --- |
| REQUIRED | 부모/자식이 같은 트랜잭션 공유 | pid 동일 — 확인됨 |
| REQUIRES_NEW | 별도 물리 트랜잭션, 부모 롤백과 무관 | pid 다름, 부모 롤백돼도 자식 생존 — 확인됨 |
| NESTED (JPA) | 설정만 손보면 될 것 | HibernateJpaDialect 구조상 원천적으로 불가능 |
| NESTED (JDBC) | savepoint까지만 롤백 | 확인됨 + 부모 최종 롤백 시 NESTED 자식도 함께 사라짐 |
| self-invocation | @Transactional 무시됨 | 확인됨 — 예외 없이 저장까지 완료됨 |
| checked exception | 기본 롤백 안 됨 | 확인됨 |

## 한계 및 남는 궁금증

- NESTED 전용 커스텀 `JpaDialect`(Hibernate 세션에서 JDBC 커넥션을 직접 꺼내 `SavepointManager`를
  구현)는 만들지 않았습니다. 실무에서 NESTED를 JPA와 결합해야 할 필요성이 크지 않다고 판단해 이번
  범위에서 제외했습니다.
- self-invocation 상태에서 저장이 이루어질 때 Hibernate 세션이 정확히 어떤 방식으로 동작하는지(순수
  autocommit인지, flush마다 트랜잭션을 암묵적으로 열고 닫는지)는 추가로 확인하지 않았습니다.

## 참고 자료

- [transaction-propagation-lab (GitHub)](https://github.com/yoonxjoong/transaction-propagation-lab)
- [스프링 트랜잭션 개념 정리 글](/posts/spring-transaction-propagation-isolation/)
- [Spring Framework Reference — Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
