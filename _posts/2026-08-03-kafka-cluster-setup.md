---
title: "카프카 클러스터 구축 실습 — Zookeeper + 브로커 3대"
description: docker-compose로 Zookeeper 기반 카프카 브로커 3대를 직접 띄우고, 토픽을 만들어 Leader/Replicas/ISR이 실제로 어떻게 배치되는지, 프로듀서/컨슈머 처리량은 어느 정도 나오는지 확인해본 실습 노트입니다.
author: yoonxjoong
date: 2026-08-03 10:00:00 +0900
categories:
  - Backend
tags:
  - Kafka
  - Docker
---

## 카프카 구성: Zookeeper(관리자) + Broker(일꾼)

카프카 클러스터는 크게 두 역할로 나뉩니다.

- **Zookeeper**: 브로커들을 관리하고 상태를 체크하는 관리자 역할
- **Broker**: 실제로 데이터를 저장하고 처리하는 일꾼 역할

브로커 3대짜리 클러스터를 docker-compose로 직접 띄워봤습니다.

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-1
    depends_on:
      - zookeeper
    ports:
      - "9091:9091"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29092,PLAINTEXT_HOST://localhost:9091
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3

  kafka-2:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-2
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3

  kafka-3:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-3
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:29092,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
```

`KAFKA_LISTENERS`는 따로 지정하지 않았는데, Confluent 이미지가 `KAFKA_ADVERTISED_LISTENERS`의 호스트명을 `0.0.0.0`으로 바꿔서 자동으로 유추해줍니다 — kafka-1은 컨테이너 안에서 `0.0.0.0:29092`(브로커 간 통신용)와 `0.0.0.0:9091`(호스트 접속용)에 바인딩되고, `ports: "9091:9091"` 매핑과 맞물려 정상 동작합니다.

## 클러스터 가동

```bash
docker compose up -d
docker compose ps
```

브로커 3개가 정상 등록됐는지는 Zookeeper에서 확인할 수 있습니다.

```bash
docker exec -it zookeeper zookeeper-shell localhost:2181 ls /brokers/ids
```

```
Connecting to localhost:2181

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[1, 2, 3]
```

## 토픽 생성과 Leader / Replicas / ISR

파티션 3개, 복제 계수 3으로 토픽을 만들었습니다.

```bash
docker exec -it kafka-1 kafka-topics \
  --create \
  --topic my-first-topic \
  --partitions 3 \
  --replication-factor 3 \
  --bootstrap-server kafka-1:29092
```

메시지를 보내고 받는 것도 확인했습니다.

```bash
# 보내기
docker exec -it kafka-1 kafka-console-producer \
  --topic my-first-topic \
  --bootstrap-server kafka-1:29092

# 받기
docker exec -it kafka-1 kafka-console-consumer \
  --topic my-first-topic \
  --from-beginning \
  --bootstrap-server kafka-1:29092
```

토픽 상세 정보를 보면 파티션마다 리더/복제본이 어떻게 배치됐는지 나옵니다.

```bash
docker exec -it kafka-1 kafka-topics \
  --describe \
  --topic my-first-topic \
  --bootstrap-server kafka-1:29092
```

```
Topic: my-first-topic   TopicId: N6vqMROxTDCzi3DDzStPWw PartitionCount: 3       ReplicationFactor: 3    Configs: 
        Topic: my-first-topic   Partition: 0    Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: my-first-topic   Partition: 1    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: my-first-topic   Partition: 2    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
```

- **Leader**: 실제 읽기/쓰기가 일어나는 브로커. 파티션 0번은 브로커 2, 파티션 1번은 브로커 3, 파티션 2번은 브로커 1이 리더입니다.
- **Replicas**: 데이터가 복제되어 저장된 브로커 목록.
- **ISR(In-Sync Replicas)**: 리더와 완전히 동기화된 복제본 목록. 브로커가 3대, 복제 계수도 3이라 모든 파티션이 브로커 3대 전부에 복제되고, ISR도 Replicas와 완전히 일치하는 상태입니다.

## 프로듀서/컨슈머 성능 테스트

카프카에 내장된 성능 테스트 도구로 처리량을 확인했습니다.

```bash
# 프로듀서: 1KB 크기 메시지 10만 개
docker exec -it kafka-1 kafka-producer-perf-test \
  --topic my-first-topic \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=kafka-1:29092

# 컨슈머: 10만 개 소비
docker exec -it kafka-1 kafka-consumer-perf-test \
  --topic my-first-topic \
  --messages 100000 \
  --bootstrap-server kafka-1:29092
```

컨슈머 테스트 결과입니다.

```
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2026-05-07 07:14:57:489, 2026-05-07 07:15:00:894, 98.1396, 28.8222, 100495, 29513.9501, 3190, 215, 456.4635, 467418.6047
```

약 3.4초 동안 98MB(약 10만 건)를 소비했고, 초당 약 3만 건 처리한 셈입니다.

## 참고 자료

- [Confluent Platform Docker 이미지](https://hub.docker.com/r/confluentinc/cp-kafka)
