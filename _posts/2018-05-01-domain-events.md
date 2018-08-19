---
layout: post
title: DomainEvents의 활용
summary: DomainEvents를 도입하게 된 배경과 고민
date: 2018-05-01 00:00:01 +0900
---

## ApplicationEvent의 한계

이벤트는 불필요한 의존성을 낮추고 확장성을 높이기 위한 주요 도구 중 하나. 스프링에는 [ApplicationEvent](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationEvent.html)라는 메커니즘을 제공함. 하지만, 이 이벤트는 트랜잭션이 완료 되기 전에 얼마든지 소비될 수 있으며, [ApplicationEventPublisher](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/ApplicationEventPublisher.html)에 대한 의존성이 강제됨.

만약, 도메인 엔티티에서 이벤트를 발행하고자 한다면 어떻게 될까? 아래와 같은 코드를 생각해 볼 수 있음.

```java
@Entity
class Order {
    
    private final ApplicationEventPublisher publisher;
    
    Order(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }
    
    void complete() {
        publisher.publish(new OrderCompleted(this));
    }
}
```

하지만, 이런 코드에서 느껴지는 몇 가지 문제점이 있다.

1. 도메인 코드가 너무 구체적인 프레임웍 기술에 의존.
2. 또한, 생성자 등을 통해 의존성을 주입받아야 함. 주입 받는 것 뿐이랴. 엔티티라서 주입도 직접 해줘야 함.
3. 이벤트는 발행됐는데 트랜잭션이 실패한다면? 이를 보완하는 메커니즘은 있으나 강제되는 것은 아님. 즉, 실수의 여지가 있음.

한 가지 시나리오를 생각해 보자.

- 기존에 사용하던 엔티티에 이벤트 발행 의무가 부과된다면?
- 모든 생성자와 생성자 호출부를 수정해야 할까?
- 열심히 수정했는데, `ApplicationEventPublisher` 대신 다른 메커니즘을 활용하게 된다면?

## 생각해 볼 수 있는 대안

`ThreadLocal` 등을 활용해 이 문제를 극복하는 코드들을 종종 봄.

- VaughnVernnon 아저씨의 [IDDD](http://www.acornpub.co.kr/book/implement-ddd)<sup>Implementing Domain-Driven Design</sup> [예시 코드](https://github.com/VaughnVernon/IDDD_Samples) 중 [DomainEventPublisher](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_common/src/main/java/com/saasovation/common/domain/model/DomainEventPublisher.java)
- 최범균 님의 [DDD Start! 예시 코드](https://github.com/madvirus/ddd-start) 중 [Events](https://github.com/madvirus/ddd-start/blob/master/src/main/java/com/myshop/common/event/Events.java)

`ThreadLocal`을 사용해야 한다는 것이 부담스럽지만, 위에서 언급한 문제들이 어느 정도 해소가 됨. 충분히 좋은 대안이라고 생각. 하지만 2가지 단점이 있다.

1. 이벤트 발행에 앞서 직접 핸들러를 등록해야 함. ([여기](https://github.com/madvirus/ddd-start/blob/master/src/main/java/com/myshop/order/command/application/CancelOrderService.java#L18)와 [여기](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/test/java/com/saasovation/collaboration/domain/model/calendar/CalendarTest.java#L158)서 확인 가능)
2. 이벤트 발행 구현체를 직접 만들고 관리해야 함.

이벤트는 서로 다른 모듈 간의 결합을 제거할 수 있는 강력한 도구다. 그런데, 이벤트를 발행하는 곳에서 수신 대상을 직접 명시함으로써 결합도가 다시 높아졌다. 또한, 직접 구현체를 직접 만들고 관리하는 일은 부담스러움. 개인적으로 이런 일은 가능한 뒤로 미루는 편이다. 비록, 사용하는 도구가 아쉽더라도 말이다. 서비스가 어느 정도 성장한 뒤, 아쉬운 부분을 고도화 할 수 있는 여지는 얼마든지 있다고 생각함.

## @DomainEvents 활용

다행인지 모르겠으나, [Spring Data Ingalls 릴리즈 소식](https://spring.io/blog/2017/01/30/what-s-new-in-spring-data-release-ingalls#domain-event-publication-from-aggregate-roots)을 보면 [@DomainEvents](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/DomainEvents.html)가 소개되고 있음.

이를 사용하면 처음의 코드가 아래처럼 바뀜.

```java
@Entity
class Order {
    
    private transient Collection<Object> domainEvents;
    
    @DomainEvents
    Collection<Object> domainEvents() {
        return this.domainEvents;
    }
    
    @AfterDomainEventPublication 
    void callbackMethod() {
        this.domainEvents.clear();
    }
    
    void complete() {
        this.domainEvents.add(new OrderCompleted(this));
    }
}
```

그리고 다음은 [AbstractAggregateRoot](https://github.com/spring-projects/spring-data-commons/blob/master/src/main/java/org/springframework/data/domain/AbstractAggregateRoot.java)를 함께 사용한 모습.

```java
@Entity
class Order extends AbstractAggregateRoot {
        
    void complete() {
        registerEvent(new OrderCompleted(this));
    }
}
```

이는 앞선 방식들에 비해 아래의 이점들을 지님.

1. 구체적인 프레임웍 기술에 의존하는 수준이 낮아짐. `ApplicationEventPublisher` 대신, `@DomainEvents`와 `@AfterDomainEventPublication`를 사용.
2. 엔티티 내부의 변경만으로 이벤트 발행이 가능함.
3. 수신부는 [@TransactionalEventListener](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionalEventListener.html) 사용하므로 트랜잭션이 성공한 경우에만 이벤트 수신.
4. 이벤트 발행부에서 수신부를 등록할 필요 없음. 직접적인 의존성 X.
5. 내가 직접 코드를 만들고 관리할 필요 X.

물론 단점도 존재한다.

1. 코드량이 늘어남. `transient` 필드를 선언하고, 여기에 이벤트 등록은 물론 해제도 직접 함.
2. 늘어난 코드량 만큼 실수의 여지 증가. 가독성도 떨어짐.
3. 대안이라고 제시하는 게 `AbstractAggregateRoot` 상속.

참고로, `AbstractAggregateRoot` 상속이 문제라고 생각하는 이유는 2가지. 먼저, 비용을 따지지 않은 채 상속들이 늘어나지 않을까 우려스러움. 또한, `AbstractAggregateRoot`라는 이름이 과연 적합한가에 대해서도 의문. 이벤트의 발행이 곧 애그리거트 루트를 의미하지는 않으며, 그 반대도 마찬가지. 이 용어가 주는 오해의 여지도 있다고 생각. 애그리거트 루트라고 하니 꼭 두 개 이상의 엔티티 집합을 말하는 것 같다. 하지만 [DDDD](http://acornpub.co.kr/book/domain-driven-design-distilled)<sup>Domain-Driven Design Distilled</sup>에 따르면, 애그리거트가 반드시 2개 이상의 엔티티 집합을 가리키는 것은 아님.

## 결과

비용을 고려하지 않는다면, 단점을 모두 보완하는 코드를 직접 구현하고 관리하는 방식이 가장 좋아 보인다. 그리고 그렇게 하고 싶은 욕심도 난다. 하지만, 지금 상황에서는 부담스러움. 좀 더 서비스가 성장한다면, 이 찝찝함을 날려버릴 기회는 얼마든지 있다고 생각. 그 전까지는 '개발자로서의 쓸데 없는 고집과 오버 엔지니어링일 거야'라는 합리화로 스스로를 달래며, `@DomainEvents`를 사용하기로 결정.

현재까지는 잘 쓰고 있음. 일부 [버그](https://jira.spring.io/browse/DATACMNS-1178)가 있어 Spring Boot의 버전업도 함께 진행했음. 앞으로도 버그나 한계는 얼마든지 있을 수 있다고 생각하며, 그 땐 직접 구현을 고려해도 되지 않을까 한다.

시간이 된다면, 도메인 이벤트 사용 시 고려해야 했던 내용들도 기록해 볼 예정.
