---
title: "아웃박스 패턴 프로젝트 (Day 2): 주문 생성과 아웃박스 기록, 한 번에!"
description: 
author: yoonxjoong
date: 2025-06-25 09:00:00 +0900
categories:
  - Software Design
tags:
---
안녕하세요! 지난번에는 **아웃박스 패턴**을 활용하여 데이터 유실 없는 이벤트 기반 아키텍처를 구축하는 프로젝트의 전체적인 그림을 그려봤었죠. 오늘은 그 대장정의 첫걸음, **Day 1: 기본 프로젝트 구조 설정 및 도메인 모델링**을 함께 진행해 보겠습니다.

단단한 기반이 있어야 튼튼한 집을 지을 수 있듯이, 프로젝트의 초기 세팅과 핵심 도메인 모델링은 매우 중요합니다. 너무 복잡하게 생각하지 말고, 필요한 것들부터 하나씩 차근차근 시작해 봅시다

---

### 프로젝트 초기 설정 : Spring Boot & Gradle

spring boot를 먼저 생성해볼게요. 저는 인텔리제이의 Spring Initializer을 활용해서 아주 쉽게 의존성들을 추가하여 프로젝트를 만들어 보겠습니다.

**Spring Initializr 설정:**

- **Project:** Gradle Project
    
- **Language:** Java
    
- **Spring Boot:** 3.5.4
    
- **Group:** `com.example.outbox`
    
- **Artifact:** `order-service`
    
- **Packaging:** Jar
    
- **Java:** 17

**필수 의존성 (Dependencies):**

- **Spring Web:** 웹 애플리케이션 개발을 위한 기본
    
- **Spring Data JPA:** JPA와 데이터베이스 연동
    
- **PostgreSQL Driver:** PostgreSQL 데이터베이스 연결 드라이버
    
- **Lombok:** 반복적인 코드(Getter, Setter 등)를 줄여주는 유용한 라이브러리
    
- **Spring for Apache Kafka:** Kafka 연동 (나중에 이벤트 발행 및 수신에 사용)

---

### 2. `application.properties`설정

PostgreSQL 데이터베이스와 JPA 연동을 위한 기본 설정을 추가해야 합니다. `src/main/resources/application.properties` 파일을 열고 다음 내용을 추가해 주세요. 

application.properties

```
# DataSource Settings (PostgreSQL)
spring.datasource.url=jdbc:postgresql://localhost:5432/outbox_db
spring.datasource.username=postgres
spring.datasource.password=your_password # 실제 비밀번호로 변경하세요!
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA Settings
spring.jpa.hibernate.ddl-auto=update # 개발 단계에서는 'update'로 테이블 자동 생성/수정
spring.jpa.show-sql=true # 실행되는 SQL 쿼리 로그 출력
spring.jpa.properties.hibernate.format_sql=true # SQL 쿼리 포맷팅

# Kafka Settings (추후 사용)
spring.kafka.bootstrap-servers=localhost:9092 # 로컬 Kafka 브로커 주소
spring.kafka.consumer.group-id=order_service_group
spring.kafka.producer.retries=3 # 메시지 전송 실패 시 재시도 횟수
```

**중요:** `spring.datasource.password`는 여러분의 PostgreSQL 비밀번호로 반드시 변경해야 합니다. `outbox_db` 데이터베이스가 아직 없다면, PostgreSQL 클라이언트(psql 등)에서 미리 생성해두는 것이 좋습니다.

SQL
```
-- PostgreSQL에서 데이터베이스 생성 (CLI)
CREATE DATABASE outbox_db;
```

### 3. 핵심 도메인 모델링: `Order` 엔티티

이제 주문 서비스를 위한 핵심 엔티티인 **`Order`**를 정의해 보겠습니다. 단순화를 위해 필수적인 필드만 포함하고, JPA 매핑을 적용합니다.

`src/main/java/com/example/outbox/order/domain/Order.java` 파일을 생성하고 다음 코드를 작성합니다.

Java

```
package com.example.outbox.order.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import lombok.Builder;
import java.time.LocalDateTime;
import java.util.UUID; // 주문 ID를 UUID로 사용

@Entity
@Table(name = "orders") // 'order'는 SQL 예약어일 수 있으므로 'orders'로 변경
@Getter
@Setter // 엔티티에 @Setter는 지양하지만, 예시의 편의를 위해 포함. 실제 프로덕션에서는 Builder나 메서드로 제어 권장
@NoArgsConstructor // JPA를 위한 기본 생성자
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID) // UUID 생성 전략 사용 (MySQL 8.0 이상, PostgreSQL 13.0 이상)
    private UUID id;

    @Column(nullable = false)
    private Long userId; // 주문한 사용자 ID

    @Column(nullable = false)
    private String productName; // 상품 이름

    @Column(nullable = false)
    private int quantity; // 수량

    @Column(nullable = false)
    private double totalPrice; // 총 가격

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status; // 주문 상태 (예: PENDING, PAID, SHIPPED, CANCELLED)

    @Column(nullable = false)
    private LocalDateTime orderDate; // 주문 일시

    // Lombok의 @Builder 어노테이션 활용
    @Builder
    public Order(Long userId, String productName, int quantity, double totalPrice, OrderStatus status, LocalDateTime orderDate) {
        this.userId = userId;
        this.productName = productName;
        this.quantity = quantity;
        this.totalPrice = totalPrice;
        this.status = status;
        this.orderDate = orderDate;
    }

    // 비즈니스 로직에 따른 상태 변경 메서드 (setter 대신)
    public void markAsPaid() {
        if (this.status == OrderStatus.PENDING) {
            this.status = OrderStatus.PAID;
        } else {
            throw new IllegalStateException("주문 상태가 PENDING이 아니어서 결제 완료 처리할 수 없습니다.");
        }
    }
}
```

`OrderStatus` Enum도 정의해 줍시다. `src/main/java/com/example/outbox/order/domain/OrderStatus.java` 파일을 생성합니다.

Java

```
package com.example.outbox.order.domain;

public enum OrderStatus {
    PENDING,  // 주문 대기 (결제 전)
    PAID,     // 결제 완료
    SHIPPED,  // 배송 중
    DELIVERED,// 배송 완료
    CANCELLED // 주문 취소
}
```

---

### 📊 4. 아웃박스 테이블(`OutboxMessage`) 모델링

이제 아웃박스 패턴의 핵심인 **`OutboxMessage`** 엔티티를 정의할 차례입니다. 이 테이블은 실제 메시지 큐로 전송될 이벤트 메시지들을 임시로 저장하는 역할을 합니다.

`src/main/java/com/example/outbox/outbox/domain/OutboxMessage.java` 파일을 생성하고 다음 코드를 작성합니다.

Java

```
package com.example.outbox.outbox.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Builder;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "outbox_messages")
@Getter
@NoArgsConstructor // JPA를 위한 기본 생성자
public class OutboxMessage {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, updatable = false)
    private String aggregateType; // 이벤트가 발생한 도메인 타입 (예: "Order")

    @Column(nullable = false, updatable = false)
    private UUID aggregateId; // 이벤트가 발생한 도메인 엔티티의 ID (예: 주문 ID)

    @Column(nullable = false, updatable = false)
    private String eventType; // 이벤트 타입 (예: "OrderPaidEvent")

    @Column(nullable = false, columnDefinition = "jsonb") // JSONB 타입으로 저장 (PostgreSQL 특화)
    private String payload; // 이벤트 데이터 (JSON 문자열 형태로 저장)

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OutboxStatus status; // 메시지 처리 상태 (PENDING, SENT)

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt; // 메시지 생성 시간

    @Column
    private LocalDateTime processedAt; // 메시지 처리 시간

    @Builder
    public OutboxMessage(String aggregateType, UUID aggregateId, String eventType, String payload, OutboxStatus status, LocalDateTime createdAt) {
        this.aggregateType = aggregateType;
        this.aggregateId = aggregateId;
        this.eventType = eventType;
        this.payload = payload;
        this.status = status;
        this.createdAt = createdAt;
    }

    // 메시지 상태 변경 메서드
    public void markAsSent() {
        this.status = OutboxStatus.SENT;
        this.processedAt = LocalDateTime.now();
    }
}
```

`OutboxStatus` Enum도 정의해 줍니다. `src/main/java/com/example/outbox/outbox/domain/OutboxStatus.java` 파일을 생성합니다.

Java

```
package com.example.outbox.outbox.domain;

public enum OutboxStatus {
    PENDING, // 아직 발행되지 않은 상태
    SENT     // 발행 완료된 상태
}
```