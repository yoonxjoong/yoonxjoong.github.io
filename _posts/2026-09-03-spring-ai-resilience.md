---
title: "Spring AI로 만든 Gemini 챗봇에 Retry+CircuitBreaker+Bulkhead를 그대로 얹어보기"
description: ecommerce-msa의 payment-service 호출에 적용했던 Retry+CircuitBreaker+Bulkhead 조합을, 이번엔 결제 API가 아니라 Spring AI로 연동한 Gemini 챗봇 호출에 적용해봤습니다. 같은 패턴이 그대로 통하는 부분과, LLM 호출이라서 다르게 생각해야 하는 부분을 정리했습니다.
author: yoonxjoong
date: 2026-09-03 10:00:00 +0900
categories:
  - Backend
  - Spring
tags:
  - Spring AI
  - Gemini API
  - Resilience4j
  - Circuit Breaker
  - MSA
mermaid: true
---

> [지난 글](/posts/webflux-nonblocking-gemini-api/)에서는 Gemini API를 WebClient로 직접 다루면서 논블로킹 체인을 손으로 짰습니다. 이번엔 반대로 **Spring AI**라는 상위 추상화를 써서 같은 종류의 외부 API 연동을 다시 해보고, 여기에 [ecommerce-msa](https://github.com/yoonxjoong/ecommerce-msa)의 결제 API 호출에 적용했던 Retry+CircuitBreaker+Bulkhead 조합을 그대로 얹어봤습니다. 코드는 [spring-ai-resilience-lab](https://github.com/yoonxjoong/spring-ai-resilience-lab)에 있습니다.

## Spring AI가 대신 해주는 것

지난 글에서 WebClient로 Gemini를 직접 호출할 땐 요청 바디 구성, 응답 파싱, 논블로킹 체인 조립을 전부 손으로 짰습니다. Spring AI는 이 과정을 `ChatClient`라는 하나의 인터페이스 뒤로 감춥니다.

```java
@Service
public class ResilientChatService {

    private final ChatClient chatClient;

    public ResilientChatService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    public String chat(String prompt) {
        return chatClient.prompt(prompt).call().content();
    }
}
```

`application.yml`에 API 키와 모델만 지정하면 나머지는 auto-configuration이 처리합니다.

```yaml
spring:
  ai:
    google:
      genai:
        api-key: ${GEMINI_API_KEY}
        chat:
          model: gemini-2.5-flash
```

`ChatClient`가 내부적으로 HTTP 호출과 응답 파싱을 감춰주는 건 편하지만, 동시에 **"외부 API 호출"이라는 사실 자체가 흐릿해지기 쉽다**는 뜻이기도 합니다. `chatClient.prompt(...).call()`은 그냥 메서드 호출처럼 보이지만, 그 안에서는 여전히 네트워크 너머의 Gemini 서버를 부르고 있습니다. 그래서 여기에도 payment-service 호출과 똑같이 회복탄력성 계층을 씌워야 합니다.

## Retry + CircuitBreaker + Bulkhead 그대로 적용

`payment-service` 호출에 붙였던 것과 동일한 구조를 그대로 씁니다. fallback을 가장 바깥쪽 애노테이션(`@Retry`)에 둬야 하는 이유는 [Circuit Breaker 글](/posts/circuit-breaker-resilience4j/)에서 다뤘던 것과 완전히 동일합니다 — `@CircuitBreaker`에 fallback을 두면 실패가 예외 대신 값으로 삼켜져서, 바깥의 `@Retry`가 재시도할 기회 자체를 못 얻습니다.

```java
@Retry(name = "geminiChat", fallbackMethod = "chatFallback")
@CircuitBreaker(name = "geminiChat")
@Bulkhead(name = "geminiChat")
public ChatResponseDto chat(String prompt, boolean simulateFailure) {
    if (simulateFailure) {
        throw new GeminiCallException("Gemini 호출 실패를 의도적으로 재현함");
    }
    String answer = chatClient.prompt(prompt).call().content();
    return new ChatResponseDto(answer, false);
}

private ChatResponseDto chatFallback(String prompt, boolean simulateFailure, Throwable t) {
    log.warn("Gemini 호출 실패(재시도+서킷브레이커 소진), cause={}", t.toString());
    return new ChatResponseDto("일시적으로 답변을 생성할 수 없습니다. 잠시 후 다시 시도해주세요.", true);
}
```

`simulateFailure` 플래그로 진짜 Gemini 호출 없이도 장애 상황을 켜고 끌 수 있게 한 것도 `payment-service`의 `PaymentClient`와 같은 패턴입니다. 결제 API든 LLM API든, "외부 호출이 실패한다"는 조건을 코드 레벨에서 강제로 만들 수 있어야 Retry/CircuitBreaker가 실제로 동작하는지 반복해서 확인할 수 있습니다.

## 여기까진 똑같은데, LLM 호출이라서 다르게 생각해야 하는 부분

패턴 자체는 그대로 옮겨 붙였지만, 옮기면서 "이건 결제 API랑 다르게 봐야겠다" 싶은 지점이 몇 개 보였습니다.

**응답 시간의 편차가 훨씬 큽니다.** 결제 API는 성공이든 실패든 응답 시간이 어느 정도 일정한 편이라, CircuitBreaker의 실패율 임계치를 설계할 때 "느려서 실패로 잡히는" 경우와 "진짜 죽어서 실패로 잡히는" 경우를 어느 정도 구분해서 생각할 수 있습니다. LLM 응답은 요청마다 생성해야 하는 토큰 수가 다르기 때문에, 같은 서버가 멀쩡한 상태에서도 응답 시간이 몇 배씩 차이 날 수 있습니다. 타임아웃을 payment-service 기준(3초)처럼 짧게 잡으면 정상적으로 느린 응답까지 실패로 잡아서 CircuitBreaker를 불필요하게 열어버릴 수 있습니다.

**재시도가 결제 API보다 훨씬 비쌉니다.** `payment-service`는 Idempotency Key로 중복 승인을 막아뒀기 때문에 재시도해도 안전합니다. LLM 호출은 애초에 "같은 질문을 다시 던져서 같은 답을 받는다"는 멱등성 개념 자체가 결제만큼 명확하지 않고, 매 시도가 각각 과금 대상인 토큰을 소비합니다. Retry가 3번 도는 게 결제 API에서는 "같은 요청을 3번 확인해보는" 정도지만, LLM 호출에서는 "돈을 3번 쓰는" 것에 더 가깝습니다.

**"실패"의 정의가 더 흐릿합니다.** 결제 API는 HTTP 상태 코드나 응답 필드로 성공/실패가 명확하게 갈립니다. LLM 응답은 HTTP 레벨에서는 200이 왔는데 내용이 원하는 형식이 아니거나(예: JSON을 요청했는데 자연어로 답함), 답변 자체가 비어있는 경우처럼 애매한 실패가 있을 수 있습니다. 이런 경우까지 CircuitBreaker가 잡아내게 하려면, HTTP 예외뿐 아니라 애플리케이션 레벨에서 응답을 검증하고 실패로 변환하는 로직이 따로 필요합니다 — 이번 랩에서는 여기까지는 다루지 않았습니다.

## 참고 자료

- [Spring AI 공식 문서 — Google GenAI Chat](https://docs.spring.io/spring-ai/reference/api/chat/google-genai-chat.html)
- [Circuit Breaker: 장애가 옆으로 안 번지게 막는 법](/posts/circuit-breaker-resilience4j/) — fallback 위치 함정 설명
- [Gemini API 연동 전에 WebFlux로 먼저 만들어본 테스트 프로젝트](/posts/webflux-nonblocking-gemini-api/) — 같은 Gemini API를 WebClient로 직접 다룬 이전 글
- [spring-ai-resilience-lab](https://github.com/yoonxjoong/spring-ai-resilience-lab) — 이 글의 코드
- [ecommerce-msa](https://github.com/yoonxjoong/ecommerce-msa) — Retry+CircuitBreaker+Bulkhead 조합의 원형
