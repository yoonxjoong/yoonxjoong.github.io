---
title: "아웃박스 패턴 프로젝트 (Day 3): 아웃박스 이벤트, Kafka로 발행하기!"
description: 
author: yoonxjoong
date: 2025-08-06 09:00:00 +0900
categories:
  - Software Design
tags:
---
## 아웃박스 패턴 프로젝트 (Day 3): 아웃박스 이벤트, Kafka로 발행하기!

안녕하세요! 아웃박스 패턴 프로젝트의 셋째 날입니다. 지난 Day 2에서는 주문 생성과 동시에 아웃박스 테이블에 이벤트 메시지를 원자적으로 기록하는 핵심 로직을 구현했죠. 이제 아웃박스 테이블에 안전하게 저장된 이벤트 메시지들을 실제로 읽어와 **Kafka 메시지 큐로 발행하는 역할**을 구현할 차례입니다.

이 역할을 담당하는 것이 바로 **'이벤트 릴레이(Event Relay)'** 또는 **'폴링 퍼블리셔(Polling Publisher)'**입니다. 우리는 간단한 스케줄러를 사용하여 아웃박스 테이블을 주기적으로 확인하고, 발행 대기 중인 이벤트를 찾아 Kafka로 전송하는 방식으로 구현해 보겠습니다.

---

### 1. 이벤트 릴레이(Polling Publisher) 서비스 생성

가장 먼저 할 일은 아웃박스 메시지를 조회하고 Kafka로 발행하는 로직을 담을 서비스를 만드는 것입니다. Spring의 `@Scheduled` 어노테이션을 활용하여 특정 주기로 이 메서드를 실행하도록 설정할 겁니다.

`
```Java
package com.example.outbox.orderservice.service;  
  
import com.example.outbox.orderservice.domain.OutboxMessage;  
import com.example.outbox.orderservice.domain.OutboxStatus;  
import com.example.outbox.orderservice.infra.OutboxMessageRepository;  
import java.util.List;  
import lombok.RequiredArgsConstructor;  
  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.kafka.core.KafkaTemplate;  
import org.springframework.scheduling.annotation.Scheduled;  
import org.springframework.stereotype.Service;  
import org.springframework.transaction.annotation.Transactional;  
  
@Service  
@RequiredArgsConstructor  
@Slf4j  
public class OutboxMessageRelayService {  
    private final OutboxMessageRepository outboxMessageRepository;  
    private final KafkaTemplate<String, String> kafkaTemplate; // Kafka 메시지 발행을 위한 템플릿  
  
    // Kafka 토픽 이름    private static final String OUTBOX_TOPIC = "outbox.events";  
  
    /**  
     * 주기적으로 아웃박스 테이블을 스캔하여 발행 대기 중인 이벤트를 Kafka로 발행합니다.     * 이 메서드는 별도의 트랜잭션으로 실행되어야 합니다.     */    @Scheduled(fixedDelay = 5000)  
    @Transactional  
    public void publishOutboxMessages() {  
        // 발행 대기 중인 메시지 조회  
        List<OutboxMessage> pendingMessages = outboxMessageRepository.findByStatusOrderByCreatedAtAsc(  
                OutboxStatus.PENDING);  
  
        if (pendingMessages.isEmpty()) {  
            log.info("발행 대기 중인 아웃박스 메시지가 없습니다.");  
            return;  
        }  
  
        System.out.println("✨ 발행 대기 중인 아웃박스 메시지 " + pendingMessages.size() + "건 발견!");  
  
        for (OutboxMessage message : pendingMessages) {  
            try {  
                // Kafka로 메시지 발행 (토픽, 메시지 키, 메시지 값)  
                // 메시지 키는 aggregateId를 사용하여 파티션 순서 보장에 도움을 줄 수 있음                kafkaTemplate.send(OUTBOX_TOPIC, message.getAggregateId().toString(), message.getPayload())  
                        .whenComplete((result, ex) -> {  
                            if (ex == null) {  
                                log.info("Kafka 발행 성공: 메시지 ID " + message.getId());  
                                message.markAsSent();  
                                outboxMessageRepository.save(message);  
                            } else {  
                                log.warn("Kafka 발행 실패: 메시지 ID " + message.getId() + ", 오류: " + ex.getMessage());  
                            }  
                        });  
            } catch (Exception e) {  
                log.error("아웃박스 메시지 발행 중 예상치 못한 오류 발생: " + message.getId() + ", 오류: " + e.getMessage());  
            }  
        }  
    }  
}
```

---

### 2. `OutboxMessageRepository`에 조회 메서드 추가

`OutboxMessageRelayService`에서 사용할 `findByStatusOrderByCreatedAtAsc` 메서드를 `OutboxMessageRepository`에 추가해야 합니다.

```java
package com.example.outbox.orderservice.infra;  
  
import com.example.outbox.orderservice.domain.OutboxMessage;  
import com.example.outbox.orderservice.domain.OutboxStatus;  
import java.util.List;  
import java.util.UUID;  
import org.springframework.data.jpa.repository.JpaRepository;  
  
public interface OutboxMessageRepository extends JpaRepository<OutboxMessage, UUID> {  
    // 상태가 PENDING인 메시지들을 생성 시간 오른차순으로 조회  
    List<OutboxMessage> findByStatusOrderByCreatedAtAsc(OutboxStatus status);  
  
}
```

---

### 3. Spring Scheduler 활성화

`@Scheduled` 어노테이션이 작동하도록 Spring Boot 애플리케이션의 메인 클래스에 `@EnableScheduling` 어노테이션을 추가해야 합니다.

`src/main/java/com/example/outbox/OrderServiceApplication.java` 파일을 수정합니다.



```Java
package com.example.outbox.orderservice;  
  
import org.springframework.boot.SpringApplication;  
import org.springframework.boot.autoconfigure.SpringBootApplication;  
import org.springframework.scheduling.annotation.EnableScheduling;  
  
@SpringBootApplication  
@EnableScheduling  
public class OrderServiceApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(OrderServiceApplication.class, args);  
    }  
  
}
```

---

### 4. Kafka 설정 추가 (생산자)

Kafka 메시지 발행을 위한 설정은 `application.properties`에 이미 추가했지만, `KafkaTemplate`을 사용하기 위한 기본적인 의존성이 `pom.xml` (Maven) 또는 `build.gradle` (Gradle)에 있는지 확인해야 합니다.

**`build.gradle` 확인:**

```bash 
dependencies {
    // ... 기존 의존성 ...
    implementation 'org.springframework.kafka:spring-kafka' // 이 의존성이 있어야 합니다.
    // ...
}
```

`KafkaTemplate`은 Spring Boot의 자동 설정 덕분에 별도의 빈(Bean) 설정을 하지 않아도 주입받아 바로 사용할 수 있습니다.

---

###  테스트 방법

이제 애플리케이션을 실행하고, Day 2에서 만들었던 주문 생성 API를 호출해 보세요.

1. **Kafka 브로커 실행:** 로컬에 Kafka가 실행 중인지 확인하세요. (Docker 등으로 쉽게 띄울 수 있습니다)
    
2. **OrderServiceApplication 실행:** Spring Boot 애플리케이션을 실행합니다.
    
3. **POST 요청 보내기:** Postman, Insomnia, curl 등으로 `/api/orders` 엔드포인트에 `POST` 요청을 보냅니다.

    ```
    POST http://localhost:8080/api/orders
    Content-Type: application/json
    
    {
        "userId": 1,
        "productName": "Awesome Gadget",
        "quantity": 2,
        "totalPrice": 199.98
    }
    ```

**예상 결과:**

- 콘솔에 `주문이 성공적으로 저장되었습니다.` 메시지가 보입니다.
- `아웃박스 메시지가 성공적으로 저장되었습니다.` 메시지도 보입니다.
- 그리고 5초(설정된 `fixedDelay` 값) 이내에 `OutboxMessageRelayService`가 실행되면서, `발행 대기 중인 아웃박스 메시지 ~건 발견!` 메시지가 보이고, 이어서 `Kafka 발행 성공` 메시지가 나타날 겁니다.
- 데이터베이스의 `outbox_messages` 테이블을 확인해 보면, 해당 메시지의 `status`가 `PENDING`에서 `SENT`로 변경되어 있을 겁니다.
- Kafka 컨슈머(`kafka-console-consumer` 등)로 `outbox.events` 토픽을 구독하면 발행된 JSON 메시지를 확인할 수 있습니다.
    ```bash
    # Kafka 컨슈머 실행 예시 (로컬 Kafka 기준)
    kafka-console-consumer --bootstrap-server localhost:9092 --topic outbox.events --from-beginning
    ```
    
---

### 마무리하며

오늘 우리는 아웃박스 테이블에 저장된 이벤트 메시지를 Kafka로 발행하는 '이벤트 릴레이'를 구현했습니다. 이제 주문 서비스는 자신의 데이터베이스와만 통신하고, 이벤트 발행의 복잡한 부분은 아웃박스 패턴과 이벤트 릴레이가 담당하게 되었습니다.

이로써 주문 서비스는 데이터 유실 없이 '주문 생성' 비즈니스 이벤트가 성공적으로 외부로 전파될 수 있다는 강력한 보증을 얻게 됩니다.

궁금한 점이 있다면 언제든지 질문해주세요! 다음 글에서 만나요!