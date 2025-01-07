---
title: Spring Boot의 @ComponentScan 이해하기
description: Spring Boot 애플리케이션에서 @ComponentScan을 사용하여 빈(Bean)을 자동으로 스캔하고 관리하는 방법을 알아봅니다.
author: yoonxjoong
date: 2025-01-07 14:00:00 +0900
categories: [Spring, Spring Boot]
tags: [Spring Boot, ComponentScan, Java]
---
# Spring Boot의 @ComponentScan 이해하기

Spring Boot 애플리케이션을 개발하다 보면, 특정 패키지 내에 있는 빈(Bean)을 자동으로 스캔하여 컨테이너에 등록하는 과정을 자주 마주하게 됩니다. 이 과정은 `@ComponentScan`이라는 어노테이션을 통해 이루어집니다. 이번 글에서는 `@ComponentScan`의 동작 원리와 사용 방법에 대해 자세히 알아보겠습니다.

---

## @ComponentScan이란?

`@ComponentScan`은 Spring Framework에서 제공하는 어노테이션으로, 지정된 패키지를 스캔하여 `@Component`, `@Service`, `@Repository`, `@Controller` 등으로 정의된 클래스를 찾아 Spring 컨테이너에 등록합니다. 이를 통해 애플리케이션은 빈을 자동으로 관리하고 의존성을 주입할 수 있습니다.

Spring Boot에서는 기본적으로 `@SpringBootApplication` 어노테이션이 포함된 클래스의 패키지와 하위 패키지를 자동으로 스캔합니다. 이는 `@ComponentScan`이 내부적으로 사용되기 때문입니다.

---

## @ComponentScan 사용 방법

### 기본 사용법

```java
@ComponentScan("com.example.package")
public class AppConfig {
    // 애플리케이션 설정 클래스
}
```

위 예제는 `com.example.package` 패키지를 스캔하여 해당 패키지 내의 빈을 등록합니다.

Spring Boot에서 `@SpringBootApplication` 어노테이션이 다음과 같이 선언되어 있습니다:

```java
@SpringBootApplication
@ComponentScan
public @interface SpringBootApplication {
    // 생략
}
```

따라서 `@SpringBootApplication`이 선언된 클래스가 위치한 패키지를 기준으로 자동 스캔이 이루어집니다.

### 특정 패키지 스캔 설정

`@ComponentScan`을 사용하여 스캔할 패키지를 명시적으로 지정할 수도 있습니다.

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.example.service", "com.example.repository"})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

위 코드는 `com.example.service`와 `com.example.repository` 패키지를 스캔하여 빈을 등록합니다.

### 특정 클래스에서 시작점 지정

`@ComponentScan`을 사용할 때 `basePackageClasses` 속성을 이용하여 스캔의 기준이 될 클래스를 지정할 수 있습니다.

```java
@ComponentScan(basePackageClasses = MyComponent.class)
public class AppConfig {
    // MyComponent 클래스가 속한 패키지부터 스캔
}
```

이 방법은 클래스 기반으로 스캔 범위를 지정할 때 유용합니다.

---

## @ComponentScan에서 제외하기

스캔 대상에서 특정 클래스를 제외하려면 `excludeFilters` 속성을 사용할 수 있습니다.

```java
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = {ExcludedComponent.class})
)
public class AppConfig {
}
```

위 예제는 `ExcludedComponent` 클래스를 스캔에서 제외합니다.

---

## 주의사항

1. **패키지 구조 관리**: `@SpringBootApplication`이 위치한 패키지의 상위 패키지에 다른 구성 요소가 있다면 스캔되지 않을 수 있으므로 패키지 구조를 신중히 설계해야 합니다.
2. **스캔 범위의 최적화**: 불필요한 패키지를 스캔하면 성능에 영향을 줄 수 있으므로 정확한 범위를 지정하는 것이 중요합니다.

---

## 결론

`@ComponentScan`은 Spring Boot 애플리케이션에서 빈을 관리하고 의존성을 주입하는 데 핵심적인 역할을 합니다. 기본적으로 `@SpringBootApplication`에 포함되어 자동으로 동작하지만, 필요에 따라 스캔 범위를 명시적으로 지정하거나 특정 클래스를 제외할 수도 있습니다.

애플리케이션의 구조와 요구 사항에 맞게 `@ComponentScan`을 활용하면 보다 효율적이고 유연한 Spring Boot 애플리케이션을 개발할 수 있습니다.

