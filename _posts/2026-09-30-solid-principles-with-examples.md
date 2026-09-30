---
title: "SOLID 원칙 정리 — 예시로 보는 SRP, OCP, LSP, ISP, DIP"
description: "객체지향 설계 5원칙(SOLID)을 각각 나쁜 예제와 개선한 예제로 비교하며 정리했습니다. 특히 의존관계 역전 원칙(DIP)은 인터페이스 설계와 생성자 주입까지 자세히 다뤘습니다."
author: yoonxjoong
date: 2026-09-30 11:00:00 +0900
categories:
  - Backend
tags:
  - SOLID
  - OOP
  - Java
mermaid: false
---

SOLID는 객체지향 설계에서 변경에 강하고 재사용하기 쉬운 코드를 만들기 위한 다섯 가지 원칙입니다.
각 원칙마다 지키지 않았을 때 생기는 문제를 코드로 먼저 보고, 그 문제를 어떻게 고치는지 순서로
정리했습니다.

## SRP — 단일 책임 원칙

하나의 클래스는 하나의 책임만 가져야 합니다. 여기서 "책임"은 "변경되는 이유"를 뜻합니다 — 서로 다른
이유로 변경될 수 있는 기능이 한 클래스에 섞여 있으면, 한쪽을 고치다가 다른 쪽이 망가질 위험이 커집니다.

**나쁜 예**

```java
class ReportGenerator {

    void generate(SalesData data) {
        String content = buildContent(data);       // 데이터 가공
        saveToFile(content);                        // 파일 저장
        sendEmail(content);                         // 이메일 발송
    }

    private String buildContent(SalesData data) { /* ... */ return ""; }
    private void saveToFile(String content) { /* ... */ }
    private void sendEmail(String content) { /* ... */ }
}
```

`ReportGenerator`는 데이터 가공 로직이 바뀔 때도, 저장 방식(파일 → DB)이 바뀔 때도, 발송 방식(이메일
→ 슬랙)이 바뀔 때도 전부 수정 대상이 됩니다. 서로 관련 없는 세 가지 변경 이유가 한 클래스에 묶여
있습니다.

**개선한 예**

```java
class ReportContentBuilder {
    String build(SalesData data) { /* ... */ return ""; }
}

class ReportFileWriter {
    void save(String content) { /* ... */ }
}

class ReportMailer {
    void send(String content) { /* ... */ }
}
```

세 클래스로 나누면 저장 방식을 DB로 바꿔도 `ReportFileWriter`만 수정하면 되고, 나머지 두 클래스는
전혀 영향받지 않습니다.

**"책임"을 판단하는 기준**: 로버트 마틴은 책임을 "변경되는 이유"로 정의하고, 서로 다른 액터(요구사항을
요청하는 주체)를 위해 바뀌는 코드가 한 클래스에 섞여 있으면 책임이 여러 개라고 봤습니다. 위
`ReportGenerator`에서 데이터 가공 로직은 기획 쪽에서 리포트 형식을 바꿔달라고 할 때 바뀌고, 저장
로직은 인프라 쪽에서 저장소를 DB로 바꾸자고 할 때 바뀌고, 발송 로직은 운영 쪽에서 알림 채널을 바꾸자고
할 때 바뀝니다. 서로 다른 세 액터가 각자 다른 이유로 같은 클래스를 건드리게 되고, 한 요구사항 때문에
수정한 코드가 다른 액터의 기능에 의도치 않은 영향을 줄 위험이 커집니다.

SRP는 "메서드를 짧게 쪼개라"는 원칙이 아닙니다. 메서드가 길어도 그 안의 코드가 전부 같은 이유로만
바뀐다면 SRP를 위반한 게 아니고, 반대로 메서드가 짧아도 서로 다른 이유로 바뀌는 코드가 한 클래스
안에 섞여 있다면 위반입니다. 판단 기준은 코드의 길이가 아니라 변경 이유의 개수입니다.

## OCP — 개방-폐쇄 원칙

기능을 확장할 때는 기존 코드를 열어서(수정해서) 확장하는 게 아니라, 새 코드를 추가하는 것만으로
확장할 수 있어야 합니다.

**나쁜 예**

```java
class PaymentService {

    void pay(String type, int amount) {
        if (type.equals("CARD")) {
            // 카드 결제 로직
        } else if (type.equals("CASH")) {
            // 현금 결제 로직
        }
        // 결제 수단이 추가될 때마다 이 메서드에 else if가 계속 늘어남
    }
}
```

카카오페이, 토스페이 같은 결제 수단이 추가될 때마다 `PaymentService.pay()`를 열어서 분기를 추가해야
합니다. 기존에 잘 동작하던 카드/현금 결제 로직까지 같은 메서드 안에서 다시 건드릴 위험이 생깁니다.

**개선한 예**

```java
interface PaymentMethod {
    void pay(int amount);
}

class CardPayment implements PaymentMethod {
    public void pay(int amount) { /* 카드 결제 로직 */ }
}

class CashPayment implements PaymentMethod {
    public void pay(int amount) { /* 현금 결제 로직 */ }
}

class PaymentService {
    void pay(PaymentMethod method, int amount) {
        method.pay(amount);
    }
}
```

새 결제 수단이 필요하면 `PaymentMethod`를 구현하는 클래스를 하나 추가하면 됩니다. `PaymentService`는
전혀 수정하지 않아도 됩니다 — 확장에는 열려 있고, 기존 코드 수정에는 닫혀 있는 상태입니다.

이 구조의 실질적인 이점은 테스트에서 드러납니다. `CardPayment`와 `CashPayment`에 대한 기존 테스트는
새 결제 수단이 추가돼도 전혀 수정할 필요가 없습니다. `PaymentService`도 이미 검증이 끝난 코드라서
다시 테스트할 필요가 없고, 새로 추가한 결제 수단 클래스만 테스트하면 됩니다.

다만 OCP를 처음부터 모든 곳에 적용하려고 하면 안 됩니다. 변경이 일어날 지점을 사전에 전부 예측해서
인터페이스로 감싸두는 건 불가능하고, 실제로는 그렇게 예측한 지점 중 상당수가 끝까지 변경되지 않습니다.
그 경우 인터페이스와 구현체가 1:1로만 존재하는 불필요한 간접 계층이 남아서 코드를 따라가기만
어려워집니다. 실무에서는 결제 수단 추가처럼 변경이 실제로 반복해서 발생한 지점에만 선별적으로
추상화를 도입하는 편이 낫습니다.

## LSP — 리스코프 치환 원칙

자식 클래스는 부모 클래스가 쓰이는 곳 어디에서든 부모 클래스 대신 넣어도 프로그램이 정상 동작해야
합니다.

**나쁜 예**

```java
class Rectangle {
    protected int width;
    protected int height;

    void setWidth(int width) { this.width = width; }
    void setHeight(int height) { this.height = height; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    @Override
    void setWidth(int width) {
        this.width = width;
        this.height = width; // 정사각형이라 너비를 바꾸면 높이도 같이 바뀜
    }

    @Override
    void setHeight(int height) {
        this.width = height;
        this.height = height;
    }
}
```

```java
void resize(Rectangle rectangle) {
    rectangle.setWidth(5);
    rectangle.setHeight(4);
    // Rectangle이면 area()가 20이어야 정상
    // 하지만 Square를 넣으면 setHeight(4) 때문에 width까지 4로 바뀌어서 area()가 16이 됨
}
```

`Square`는 `Rectangle`을 상속했지만, `Rectangle`을 전제로 짠 코드(`resize`)에 넣으면 기대와 다른
결과가 나옵니다. `Square`가 `Rectangle`의 행동 규약(너비와 높이는 독립적으로 바뀐다)을 깨뜨리기
때문입니다.

**개선한 예**

```java
interface Shape {
    int area();
}

class Rectangle implements Shape {
    private final int width;
    private final int height;

    Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    public int area() { return width * height; }
}

class Square implements Shape {
    private final int side;

    Square(int side) { this.side = side; }

    public int area() { return side * side; }
}
```

`Square`가 `Rectangle`을 상속하는 대신, 둘 다 `Shape`라는 공통 인터페이스만 구현하도록 바꿨습니다.
`Rectangle`의 세부 행동(너비/높이를 각각 바꿀 수 있다)을 `Square`가 억지로 따라야 할 필요가
없어집니다.

**행동 계약 관점에서 보기**: LSP는 자식이 부모의 사전조건(precondition)을 더 강하게 요구하면 안 되고,
사후조건(postcondition)을 더 약하게 만들면 안 된다는 규칙으로 정리할 수 있습니다. `Rectangle`의
`setHeight()`는 "높이만 바뀌고 너비는 그대로 유지된다"는 사후조건을 갖고 있는데, `Square.setHeight()`는
너비까지 함께 바꿔서 이 사후조건을 약화시켰습니다. 수학적으로는 정사각형이 직사각형의 특수한 경우가
맞지만, "너비와 높이를 서로 독립적으로 바꿀 수 있다"는 행동 수준에서는 정사각형이 직사각형과 같은
방식으로 동작하지 않습니다. 상속 관계를 만들 때는 데이터 구조상의 is-a 관계가 아니라 행동 수준의
is-a 관계가 성립하는지를 확인해야 합니다.

LSP 위반은 예외 처리에서도 자주 나타납니다. 부모 클래스의 메서드가 특정 예외를 던지지 않는데 자식
클래스가 그 메서드를 오버라이드하면서 새로운 예외를 던지도록 만들면, 부모 타입만 알고 호출하는
코드는 그 예외를 예상하지 못한 채로 실행되다가 처리되지 않은 예외를 만나게 됩니다.

## ISP — 인터페이스 분리 원칙

클라이언트는 자신이 사용하지 않는 메서드에 의존하도록 강요받으면 안 됩니다. 하나의 거대한
인터페이스보다 작은 인터페이스 여러 개로 나누는 게 낫습니다.

**나쁜 예**

```java
interface Worker {
    void work();
    void eat();
}

class HumanWorker implements Worker {
    public void work() { /* ... */ }
    public void eat() { /* ... */ }
}

class RobotWorker implements Worker {
    public void work() { /* ... */ }

    public void eat() {
        throw new UnsupportedOperationException("로봇은 식사하지 않음");
    }
}
```

로봇은 밥을 먹지 않는데도 `Worker` 인터페이스 때문에 `eat()`을 구현해야 합니다. 억지로 구현한
`eat()`은 호출되면 예외를 던지는 코드일 뿐이라, `Worker` 타입으로 다루는 쪽에서 `RobotWorker`인지
모르고 `eat()`을 호출하면 런타임에 예외가 터집니다.

**개선한 예**

```java
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

class HumanWorker implements Workable, Eatable {
    public void work() { /* ... */ }
    public void eat() { /* ... */ }
}

class RobotWorker implements Workable {
    public void work() { /* ... */ }
}
```

`Workable`과 `Eatable`로 인터페이스를 나누면, `RobotWorker`는 필요한 `Workable`만 구현하면 됩니다.
불필요한 메서드를 구현하도록 강요받지 않습니다.

ISP는 SRP를 인터페이스 단위에 적용한 것으로 볼 수 있습니다. 인터페이스도 하나의 클라이언트 그룹이
필요로 하는 메서드만 모아야 하고, 서로 다른 클라이언트가 필요로 하는 메서드를 한 인터페이스에 몰아
넣으면 안 됩니다. 인터페이스가 커질수록 그 인터페이스에 의존하는 모든 구현체와 호출부가, 자신과
무관한 메서드가 추가되거나 바뀔 때도 영향권 안에 들어옵니다. `Worker`처럼 작은 예제에서는 체감이
안 되지만, 실무에서 메서드 수십 개짜리 인터페이스 하나에 여러 구현체가 물려 있는 경우 이 결합도가
실제 유지보수 비용으로 나타납니다.

## DIP — 의존관계 역전 원칙

고수준 모듈(정책을 결정하는 쪽)과 저수준 모듈(구체적인 구현)이 서로 직접 의존하지 않고, 둘 다
추상화(인터페이스)에 의존해야 합니다.

**나쁜 예**

```java
class EmailSender {
    void send(String message) { /* 이메일 발송 로직 */ }
}

class OrderService {
    private final EmailSender emailSender = new EmailSender(); // 구체 클래스에 직접 의존

    void placeOrder(Order order) {
        // 주문 처리 로직
        emailSender.send("주문 완료: " + order.getId());
    }
}
```

`OrderService`(고수준 모듈)가 `EmailSender`(저수준 모듈)를 직접 `new`로 생성해서 의존하고 있습니다.
이 코드에는 두 가지 문제가 있습니다.

1. 알림 방식을 이메일에서 문자로 바꾸려면 `OrderService` 내부 코드를 직접 수정해야 합니다.
2. `OrderService`를 테스트할 때 진짜 `EmailSender`가 실행되는 걸 막을 방법이 없습니다. 가짜
   객체(Mock)로 바꿔치기할 수 없기 때문입니다.

**개선한 예**

```java
interface NotificationSender {
    void send(String message);
}

class EmailSender implements NotificationSender {
    public void send(String message) { /* 이메일 발송 로직 */ }
}

class SmsSender implements NotificationSender {
    public void send(String message) { /* 문자 발송 로직 */ }
}

class OrderService {
    private final NotificationSender notificationSender;

    OrderService(NotificationSender notificationSender) { // 생성자로 주입받음
        this.notificationSender = notificationSender;
    }

    void placeOrder(Order order) {
        // 주문 처리 로직
        notificationSender.send("주문 완료: " + order.getId());
    }
}
```

```java
OrderService orderService = new OrderService(new EmailSender());
// 문자로 바꾸고 싶으면 OrderService 코드는 그대로 두고 생성자에 넘기는 구현체만 바꾸면 됨
OrderService smsOrderService = new OrderService(new SmsSender());
```

`OrderService`는 이제 `EmailSender`도 `SmsSender`도 모르고, `NotificationSender`라는 추상화만
알고 있습니다. `NotificationSender`를 구현(implement)하는 클래스가 무엇이든 `OrderService` 코드는
바뀌지 않습니다. 테스트할 때도 `NotificationSender`를 구현한 가짜 객체를 넘기면 실제 발송 없이
로직만 검증할 수 있습니다.

**"역전"이라는 이름이 붙은 이유**: 일반적인 절차형 사고방식으로는 `OrderService`(상위 정책)가
`EmailSender`(하위 구현)를 알고 사용하는 게 자연스러워 보입니다. DIP는 이 방향을 뒤집어서, 상위
모듈과 하위 모듈이 모두 추상화에 의존하게 만들고 하위 모듈이 그 추상화를 구현하도록 만듭니다. 의존의
방향이 "상위 → 하위"에서 "둘 다 → 추상화"로 뒤집히기 때문에 의존관계 "역전"입니다.

**DIP와 DI(의존성 주입)의 관계**: DIP는 "무엇에 의존해야 하는가"에 대한 설계 원칙이고, DI는 그
원칙을 실제 코드에서 지키기 위한 기법입니다. 위 예제에서 `OrderService`의 생성자로
`NotificationSender` 구현체를 넘겨준 것이 생성자 주입(constructor injection)입니다. 스프링에서
`@Service` 빈이 다른 빈을 필드가 아니라 생성자로 주입받도록 작성하는 습관이 바로 이 패턴을 프레임워크
차원에서 강제하는 것입니다.

**추상화는 어느 쪽에 두어야 하는가**: `NotificationSender` 인터페이스는 `EmailSender`가 속한
저수준 패키지가 아니라, `OrderService`가 속한 고수준 패키지 쪽에 정의해야 합니다. 인터페이스를
저수준 패키지에 정의해버리면 `OrderService`가 여전히 저수준 패키지를 참조해야 하고, 코드는 인터페이스를
호출하고 있으니 DIP를 지킨 것처럼 보이지만 패키지 사이의 실제 의존 방향은 바뀌지 않습니다.
`OrderService`가 필요로 하는 인터페이스를 `OrderService` 쪽에서 정의하고, `EmailSender`가 그
인터페이스를 구현하러 오는 방향이어야 저수준 모듈이 고수준 모듈에 의존하는 구조가 실제로 성립합니다.

## 정리

| 원칙 | 핵심 질문 |
| --- | --- |
| SRP | 이 클래스가 변경되는 이유가 하나뿐인가 |
| OCP | 기능을 추가할 때 기존 코드를 수정해야 하는가, 새 코드만 추가하면 되는가 |
| LSP | 자식 클래스를 부모 자리에 넣어도 기존 코드가 그대로 동작하는가 |
| ISP | 이 인터페이스를 구현하는 클래스가 쓰지도 않을 메서드를 억지로 구현하고 있는가 |
| DIP | 상위 모듈이 하위 모듈의 구체 클래스를 직접 알고 있는가, 추상화만 알고 있는가 |

## 참고 자료

- Robert C. Martin, *Agile Software Development, Principles, Patterns, and Practices*
