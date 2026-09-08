---
title: "Spring 트랜잭션 전파(Propagation) 속성 치트시트"
description: "@Transactional의 propagation 옵션 7가지를 표로 정리한 짧은 치트시트입니다. 개념 설명과 예시 코드는 이전 트랜잭션 정리 글에 있습니다."
author: yoonxjoong
date: 2026-09-08 09:00:00 +0900
categories:
  - Backend
tags:
  - Spring
  - Transaction
mermaid: false
---

> 개념 설명과 예시 코드는 [스프링 트랜잭션 개념 정리 글](/posts/spring-transaction-propagation-isolation/)에 있습니다. 이 글은 `propagation` 옵션 7가지만 빠르게 찾아보기 위한 치트시트입니다.

## 한눈에 보는 표

| 옵션 | 기존 트랜잭션 있음 | 기존 트랜잭션 없음 | 물리 트랜잭션 |
| --- | --- | --- | --- |
| **REQUIRED** (기본값) | 참여 | 새로 시작 | 참여 시 공유 |
| **REQUIRES_NEW** | 기존을 보류(suspend)하고 새로 시작 | 새로 시작 | 항상 별도 |
| **NESTED** | 부모 안에 savepoint 생성 | 새로 시작 | 부모와 동일 물리 트랜잭션 |
| SUPPORTS | 참여 | 트랜잭션 없이 실행 | 참여 시 공유 |
| NOT_SUPPORTED | 기존을 보류하고 트랜잭션 없이 실행 | 트랜잭션 없이 실행 | 없음 |
| MANDATORY | 참여 | 예외 발생 | 참여 시 공유 |
| NEVER | 예외 발생 | 트랜잭션 없이 실행 | 없음 |

## 롤백 전파 여부

| 옵션 | 자식 실패 시 부모 영향 | 부모 실패(롤백) 시 자식 영향 |
| --- | --- | --- |
| REQUIRED | 같은 트랜잭션이라 부모도 롤백 | 자식도 함께 롤백 |
| REQUIRES_NEW | 부모에 영향 없음 (독립 트랜잭션) | 자식은 이미 커밋됐다면 영향 없음 |
| NESTED | savepoint까지만 롤백, 부모는 계속 진행 가능 | 자식도 함께 롤백 (부모 물리 트랜잭션에 속함) |

## 실무 사용 빈도

- **REQUIRED**: 기본값, 대부분의 경우
- **REQUIRES_NEW**: 감사 로그, 알림 발송 등 "바깥이 실패해도 반드시 남아야 하는 작업"
- **NESTED**: JDBC savepoint 지원 필요 + JPA 환경 제약으로 실무에서 드묾
- SUPPORTS / NOT_SUPPORTED / MANDATORY / NEVER: 거의 안 씀 — 특정 아키텍처 제약을 강제하고 싶을 때만

## 참고 자료

- [스프링 트랜잭션 개념 정리 — 전파, 격리 수준, 그리고 프록시가 실제로 하는 일](/posts/spring-transaction-propagation-isolation/)
- [Spring Framework Reference — Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
