---
layout: post
title: Growing Object-Oriented Software, Guided by Tests
summary: 테스트 주도 개발로 배우는 객체 지향 설계와 원칙 책을 읽고 간단히 기록
date: 2018-05-20 00:00:01 +0900
---

## 책 소개

경력이 얼마 안 되던 시절 TDD로 많은 도움을 받았고, 더 배우고 싶어 이 책을 읽었었음. 막상 읽어 보니 TDD라기보다 OOP 책. 하지만, 실망은 커녕 더 큰 배움을 얻었던 기억. 이번에 원서로 다시 한번 읽어 봄. 여전히 좋았고 그래서 이렇게 기록까지 진행.

기록은 OOP 개념 위주로, 개인적으로 이해하기 쉬운 순서와 내용으로 재구성하여 작성함.

## An Web of Objects

>  "Object-oriented design focuses more on the communication between objects than on the objects themselves."
>
> — Alan Kay

객체 지향 설계는 객체 그 자체보다 객체간의 커뮤니케이션에 초점을 둔다. 여기서 커뮤니케이션이란, 객체들이 메시지를 주고받는 것을 가리킴. 다른 객체로부터 메시지를 수신, 이에 대한 반응으로 또 다른 객체들에게 메시지를 전달. 혹은 자신을 호출한 객체에게 결과 값을 반환하거나 예외를 던짐.

그래서 객체 지향 시스템은 협력하는 객체들의 망<sup>an web of collaborating objects</sup>.

![An web of collaborating objects](/images/2018-05-20-goosgt/web-of-objects.png)

## Communication over Classification

객체 지향을 처음 접할 때 클래스를 배우는 것이 일반적. 하지만 이런 Classification에서 벗어나 Communication으로의 관점 이동이 필요하다는 이야기. 아래 그림은 이런 차이를 잘 보여줌.

![roles and objects in game engine](/images/2018-05-20-goosgt/roles-and-objects-in-game-engine.png)

언뜻 보기에는 `Actor`, `Scenary`, `Obstacle`, `Effects` 등으로 대상을 구분하는 것이 유용해 보임. 그러나 이것은 게임 플레이어의 관점임. 구현자의 입장에서는 `Visible` 대상을 찾아 화면에 렌더링하고, 시간이 경과하고 있음을 `Animated` 대상에 알려주며, `Physical` 객체 간의 충돌을 감지해야 함. 어떤 클래스인지가 관심 있는 것이 아니라, 상황과 시간에 따라 필요로 하는 역할<sup>role</sup>을 찾고 이들에게 메시지<sup>message</sup>를 보냄.

## Separation of Concern

> “we gather together code that will change for the same reason.”

변경을 국소화 하라는 이야기. 한 가지 방법으로 관심사가 같은 것들을 한곳에 모아둘 수 있다. 반대로 이야기 하면, 서로 관심사가 다른 것들은 분리. 예컨대, 문자열을 파싱하는 것과 파싱된 결과를 해석하는 것을 서로 분리할 수도.

개인적으로는 [SRP: The Single Responsibility Principle](https://drive.google.com/file/d/0ByOwmqah_nuGNHEtcU5OekdDMkk/view)이 관심사의 분리를 잘 설명한다고 생각함.

```java
interface Modem {
    void Dial(string pno);
    void Hangup();
    void Send(char c);
    char Recv();
}
```

위 코드에서 `Dial`과 `Hangup`은 연결에 관련된 기능이고, `Send`와 `Recv`는 메시지 교환에 관련됨. 서로 다른 책임<sup>responsibility</sup>이라고 볼 수도 있음. 그렇다면 분리해야 할까?

> “That depends upon how the application is changing.” 

어플리케이션에서 연결 관련 부분만 변하거나, 메시지 교환 부분만 변한다면 분리를 고려할 수 있음. 반면, 4개의 메서드가 함께 바뀐다면 한곳에 있어도 됨. 중요한 것은 변경. 변경 시 함께 변화하는 코드들을 모아 두는 것이 중요.

## Higher Levels of Abstraction

> "The only way to deal with complexity is to avoid it, by working at higher levels of abstraction."

식당에 가면 메뉴를 보고 음식을 주문하지, 접시나 레시피, 재료를 일일이 지정하지 않는다. 프로그래밍에서도 마찬가지. 변수를 조작하고 흐름을 제어하는 대신, 컴포넌트의 선언적 조합을 고려. 

```java
moneyEditor
    .getAmountField()
    .setText(String.valueOf(money.amount()));

moneyEditor
    .getCurrencyField()
    .setText(money.currenceyCode());
```

위 코드는 책에 나온 예시. 언뜻 보기에 괜찮아 보일지도 모르겠지만, 적어도 2가지 냄새가 남.

1. 내부에 어떤 컴포넌트들이 있는지가 드러남.  `AmountField`와 `CurrencyField`가 그것.
2. 또, 이들 컴포넌트에 값이 함께 할당되어야 함이 드러남. 즉, 컴포넌트들의 상호작용이 드러남.

아래는 이 두 가지 문제를 하나씩 개선해 나간 코드.

```java
// 내부에 어떤 컴포넌트들이 있는지를 숨김.
moneyEditor.setAmountField(money.amount());
moneyEditor.setCurrencyField(money.currencyCode());

// 컴포넌트 간의 상호작용을 숨김.
moneyEditor.setValue(money);
```

일일이 필드를 찾고, 각 필드에 값을 함께 할당하는 대신, 그저 이 돈의 값을 설정하라고 명시함. How를 추상화하고 What으로 명령을 내리고 있음. 메뉴를 보고 주문하듯이 말이다. 표현력이 좋아진 것은 덤.

## Composite Simpler Than the Sum of its Parts

컴포짓 객체의 API는 그 객체가 가진 컴포넌트들의 단순 합보다 단순해야 한다는 이야기. 바로 위에서 언급한 코드도 이와 같은 맥락임. `MoneyEditor`는 `Amount`와 `Currency`를 가지고 있으며, 이들의 존재와 상호작용을 추상화하여 단지 `setValue(Money)`라는 보다 단순한 API를 제공하고 있음.

## Compose Objects to Describe System Behavior

저수준의 객체들을 빌딩 블럭처럼 사용해서 더 많은 일을 하는 객체를 만들라는 이야기. 이렇게 하면 상대적으로 적은 코드로 유연한 어플리케이션을 만들 수 있음. 예컨대, 저수준의 객체를 다양한 방식으로 조합하여 여러가지 시나리오를 지원. Domain 모델들을 사용자 요구에 따라 Application 레이어에서 조합하는 DDD가 떠오름.

> "For each scenario, we provide a different assembly of components to  build, in effect, a subsystem to plug into the rest application. Such  design are also easy to extend—just write a new plug-compatible  component and add it in."

우리가 흔히 작성하는 테스트 코드도 객체들을 조합해서 시스템의 행위를 묘사하는 하나의 사례라고 생각함.

```java
@Test
public void on_정기_구독에_연결된_모든_주문을_만료_처리해야_한다() {

    // given
    val subscriptionKey = //;
    val order1st = dummyOrder();
    val order2nd = dummyOrder();

    // and
    when(orderFinder.find(subscriptionKey))
        .thenReturn([order1st, order2nd]);

    // when
    commandService.on(SubscriptionOrderCancelCommand.of(sno));

    // then
    verify(orderFlowService).expire(order1st);
    verify(orderFlowService).expire(order2nd);
}
```

만약, 테스트 코드가 설명적이지 못하다면, 잘못된 디자인은 아닌지 고민해 봐야 할 것.

## Building Up to Higher-Level Programming

코드는 2가지 Layer로 나눌 수 있다.

1. Implementation Layer: describe how the code does it.
2. Declarative Layer: what the code will do.

구현 계층은 '어떻게'를, 선언 계층은 '무엇'을 나타냄. 전자는 빌딩 블럭으로, 후자는 시나리오로 볼 수도 있음. 서로 성격이 다르므로 코딩 스타일도 달라짐. 구현 계층에서는 전통적인 OOP 가이드 라인을 따름. 한편, 선언 계층에서는 상대적으로 유연하게 대응하며, 정적 메소드나 "Train Wreck"을 허용하기도 함.

## Tell, Don't Ask

내부 구조나 상태를 묻는 대신, 무엇(WHAT)을 하라고 말하라.

![Tell, Don't Ask](/images/2018-05-20-goosgt/tell-dont-ask.png)

위 그림은 Martin Fowler의 [TellDontAsk](https://martinfowler.com/bliki/TellDontAsk.html) 글에 나온 그림. 개인적으로 이 원칙을 가장 잘 설명하는 그림이라고 생각. 그림의 윗 부분은 어떤 행위를 하기 위해 다른 객체에게 이것 저것 묻고 있음. 내부 구조 등의 HOW가 밖으로 노출된 것. 이렇게 되면 행위가 이곳 저곳에 산재하게 됨.

아래는 이런 원칙을 무시한 코드. 출처는 C2Wiki의 [Train Wreck](http://wiki.c2.com/?TrainWreck).

```
client.GetMortgage()
    .PaymentCollection()
    .GetNextPayment()
    .ApplyPayment(300.00)
```

`client`의 내부 구성, 타입, 관계가 드러났다. 이 중 하나라도 변경 되면, 산재 되어 있는 모든 곳을 함께 수정하고 테스트 해야 함. 아래는 이를 개선한 것.

```
client.ApplyMortagePayment(300.00)
```

내부 변경에 좀 더 자유로우며, 표현력도 함께 개선되었다.

## But Sometimes Ask

언제나 말하기만 할 수는 없음. 우리는 값 객체나 컬렉션으로부터 정보를 얻고, 팩토리로부터 새로운 객체를 획득하기도 하며, 객체 검색이나 필터링을 위해 상태를 묻기도 함. 하지만, 이 경우에도 표현력을 잃지 않으면서 열차 전복을 피할 수 있다.

아래는 개인적으로 흔히 접하게 되는 전형적인 묻는 코드.

```java
product.getId().equals(salesProduct.getId()));
```

이 간단한 한 줄에도 4가지 잠재적 문제점이 존재함.

1. `id` 필드의 존재가 드러남. 이 필드가 없어진다면?
2. `id` 필드의 타입이 드러남. `Long`에서 `ProductId` 타입으로 바뀐다면?
3. `id`로 식별성을 검증한다는 사실이 드러남. 여러 필드의 조합으로 식별해야 한다면?
4. `id`가 `null`일 수 있음. 식별성을 검사하는 모든 곳에 `null` 체크 들어가게 될 수도.

## 기타

그 외에도 아래 주제들이 인상 깊었음.

- [No And's, Or's, Or But's](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#no-ands-ors-or-buts)
- [Context Independence](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#context-independence)
- [Value Types](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#value-types)
- [Ports and Adapters](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#ports-and-adapters)
- [Object Peer Stereotype](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#object-peer-stereotypes)
- [Unit Testing the Collaborating Objects](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#unit-testing-the-collaborating-objects)
- [Where Do Objects Come from?](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#where-do-objects-come-from)
- [Identify Relationships with Interface](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#identify-relationships-with-interfaces)
- [How Writing a Test First Helps the Design](https://github.com/codehumane/what-i-learned/blob/master/goosgt/README.md#how-writing-a-test-first-helps-the-design)