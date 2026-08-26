---
title: "K8s 환경에서 Kafka, Redis, Kafka UI 구축 — Spring Boot 멀티 포트 연동 트러블슈팅"
description: 로컬 K8s(Docker Desktop)에 Kafka와 Redis를 배포하고, 호스트의 Spring Boot와 클러스터 내부의 Kafka UI가 동시에 카프카에 접속하도록 Internal/External 이중 리스너를 구성한 과정을 정리합니다.
author: yoonxjoong
date: 2026-08-26 11:00:00 +0900
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kafka
  - Redis
  - Spring
---

## 문제 상황

K8s 안에 떠 있는 Kafka를 호스트(Mac)의 Spring Boot와, 같은 클러스터 안 Pod로 떠 있는
Kafka UI가 동시에 접속하려고 하면 한쪽이 연결에 실패합니다.

원인은 `KAFKA_ADVERTISED_LISTENERS`에 있습니다. 클라이언트는 브로커에 처음 접속할 때
메타데이터 응답으로 "이 주소로 접속해라"라는 advertised host를 받는데, 리스너를 하나만
두면 호스트 OS와 클러스터 내부 Pod 중 한쪽은 그 주소로 브로커를 찾을 수 없어서 연결
재시도만 반복하다 실패합니다.

해결책은 Kafka의 멀티 리스너 설정으로 내부용 포트(9092)와 외부용 포트(9094)를 완전히
분리하는 것입니다.

## 인프라 구성

### Kafka 배포 (이중 포트)

Internal(9092)과 External(9094) 리스너를 분리해서 배포합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-kafka
spec:
  ports:
  - port: 9092
    targetPort: 9092
    name: internal
  - port: 9094
    targetPort: 9094
    name: external
  selector:
    app: kafka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-kafka
spec:
  serviceName: my-kafka
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: apache/kafka:3.7.0
        ports:
        - containerPort: 9092
        - containerPort: 9093
        - containerPort: 9094
        env:
        - name: KAFKA_NODE_ID
          value: "1"
        - name: KAFKA_PROCESS_ROLES
          value: "controller,broker"
        - name: KAFKA_LISTENERS
          value: "INTERNAL://:9092,CONTROLLER://:9093,EXTERNAL://:9094"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "INTERNAL://my-kafka:9092,EXTERNAL://127.0.0.1:9094"
        - name: KAFKA_CONTROLLER_LISTENER_NAMES
          value: "CONTROLLER"
        - name: KAFKA_CONTROLLER_QUORUM_VOTERS
          value: "1@localhost:9093"
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
          value: "CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT"
        - name: KAFKA_INTER_BROKER_LISTENER_NAME
          value: "INTERNAL"
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "1"
```

핵심은 `KAFKA_ADVERTISED_LISTENERS`가 리스너별로 다른 주소를 광고한다는 점입니다.
INTERNAL은 클러스터 내부 DNS(`my-kafka:9092`)를, EXTERNAL은 포트포워딩 이후 호스트가
접근할 주소(`127.0.0.1:9094`)를 알려줍니다.

### Kafka UI 배포

Kafka UI는 K8s 내부 Pod이므로 Kafka의 Internal 포트를 바라보게 합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-ui
  template:
    metadata:
      labels:
        app: kafka-ui
    spec:
      containers:
      - name: kafka-ui
        image: provectuslabs/kafka-ui:latest
        ports:
        - containerPort: 8080
        env:
        - name: KAFKA_CLUSTERS_0_NAME
          value: "K8s-Kafka"
        - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
          value: "my-kafka:9092"
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
  selector:
    app: kafka-ui
```

`http://localhost:30080`으로 접속하면 Kafka UI가 Internal 리스너를 통해 브로커에
정상적으로 붙습니다.

### Redis 배포 (NodePort)

```bash
helm upgrade --install my-redis bitnami/redis \
  --set architecture=standalone \
  --set auth.enabled=false \
  --set master.service.type=NodePort \
  --set master.service.nodePorts.redis=30379
```

`localhost:30379`로 접속합니다.

## Spring Boot 연동

호스트(Mac)의 Spring Boot는 External 포트(9094)를 포트포워딩해서 씁니다.

```bash
kubectl port-forward svc/my-kafka 9094:9094
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 30379

  kafka:
    bootstrap-servers: 127.0.0.1:9094
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: delivery-external-api-group
      enable-auto-commit: false
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
    listener:
      ack-mode: MANUAL
```

## 정리

- 문제의 본질은 "advertised address는 리스너마다 달라야 한다"는 것 하나입니다.
- 내부 접속자(같은 클러스터 안 Pod)와 외부 접속자(호스트 OS)를 같은 포트/같은 주소로
  섞으려고 하면 반드시 한쪽이 깨집니다.
- `kubectl port-forward`는 세션이 떠 있는 동안만 유효하므로, 로컬 개발 중에는 별도
  터미널에 계속 띄워둬야 합니다.
