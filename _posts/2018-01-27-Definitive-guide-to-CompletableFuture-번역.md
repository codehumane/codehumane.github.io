---
layout: post
title: 번역. Definitive guide to CompletableFuture
summary: CompletableFuture의 핵심 기능들을 쉽게 소개하는 글을 번역함.
date:   2018-01-27 00:00:01 +0900
---

[HystrixCommand](https://netflix.github.io/Hystrix/javadoc/com/netflix/hystrix/HystrixCommand.html)를 사용하다 보니, 자연스럽게 `#toObservable`을 통해 [Observable](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Observable.html)을 사용하게 됨. 하지만, 실제로 사용하는 방식을 살펴보면, [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)로 충분해 보였음. 그러면서 `Observable`과 `CompletableFuture`의 차이가 뭔지 궁금해짐. 대략적으로만 알던 CompletableFuture를 좀 더 이해할 필요가 있었고, 이 때 [Java 8: Definitive guide to CompletableFuture](http://www.nurkiewicz.com/2013/05/java-8-definitive-guide-to.html) 문서가 도움이 됨. 필요한 부분만을 번역하여 기록해 둠. 다음 글에서는 `Observable`에 관련된 글 번역 혹은 요약 예정.

## 감싸진 값의 추출/수정

일반적으로 퓨처<sup>Future</sup>는 다른 스레드에서 실행되는 코드 조각을 가리킨다. 하지만 항상 그런 것은 아니다. 가끔은 [JMS 메시지 도착](http://www.nurkiewicz.com/2013/03/deferredresult-asynchronous-processing.html)과 같이, 곧 발생하리라 기대되는 이벤트를 나타내기도 한다. 즉, `Future<Message>`가 존재하더라도 실행 중인 비동기 작업은 없을 수 있다. 그리고 JMS 메시지가 도착하면 이 퓨처를 완료<sup>complete(resolve)</sup>시킨다. 이게 바로 이벤트 드리븐이다. 단지 `CompletableFuture`를 만들고, 클라이언트에게 전달해 준 뒤, 작업이 완료되었다고 생각되면 퓨처에 대해 `complete()`을 호출하기만 하면 된다. 퓨처의 완료를 기다리던 클라이언트는 이제 다음 작업을 계속 할 수 있다.

초보자를 위해, 간단한 `CompletableFuture`를 만들어 클라이언트에게 전달하는 코드를 살펴보자.

```java
public CompletableFuture<String> ask() {
  final CompeltableFuture<String> future = new CompletableFuture<>();
  // ...
  return future;
}
```

이 퓨처에는 `Callable<String>`도 연관되어 있지 않고, 스레드 풀이나 비동기 작업도 없다. 만약 클라이언트 코드가 `ask().get()`을 호출하면, 클라이언트는 영원히 대기하게 된다. 완료 콜백을 등록해준다고 하더라도, 영원히 실행되지 않을 것이다. 그래서 어떻게 하란 말인가?

```java
future.complete("42");
```

위 코드를 실행하면 대기중이던 모든 클라이언트의 `Future.get()` 코드가 결과 문자열을 받게 된다. 물론, 완료 콜백도 즉시 수행된다. 이는 앞으로 일어날 일(반드시 어떤 스레드에서 실행되야 하는 연산일 필요는 없다)을 표현할 때 유용하다. `CompletableFuture.complete()`은 오직 한 번만 호출될 수 있으며, 이어지는 호출들은 무시된다. 하지만 `CompletableFuture.obtrudeVale(...)`라는 뒷구멍이 존재한다. 이 함수는 이미 제공된 `Future`의 이전 값을 오버라이드 한다. 주의해서 사용해야 한다.

종종 실패를 감지하고 싶을 때가 있다. 잘 알다시피 `Future` 객체는 래핑된 결과나 예외를 다룰 수 있다. 예외를 전달하고 싶다면, `CompletableFuture.completeExceptionally(ex)`를 사용하면 된다. `completeExceptionally`가 호출되면 모든 클라이언트의 대기는 해제되고, `get()`에서 예외가 일어난다. 예외를 다루는 방식이 조금 다르긴 하지만 `CompletableFuture.join()` 메소드도 있다. 일반적으로 둘은 동일하게 취급된다. 그리고 마지막으로 `CompletableFuture.getNow(valueIfAbsent)` 메소드가 있다. 이 메소드는 클라이언트를 대기시키지 않는다. 그리고 만약, `Future`가 완료되지 않은 경우에 호출되면 기본 값을 반환한다. 너무 많이 기다리지 않아야 하는 시스템을 만들 때 유용하다.

마지막으로 `completedFuture(value)`라는 `static` 유틸리티 메소드가 있다. 이는 이미 완료된 `Future` 객체를 반환한다. 테스트 혹은 어댑터 레이어를 작성할 때 유용할 수 있다.

## CompletableFuture의 생성과 획득

자 그런데, `CompletableFuture`는 직접 생성만 가능한가? 물론 아니다. 다른 퓨처들 처럼, 이미 존재하는 작업을 `CompletableFuture`로 감쌀 수 있다. 아래의 팩토리 메소드들로 말이다.

```java
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor);
static CompletableFuture<Void> runAsync(Runnable runnable);
static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);
```

`Executor`를 인수로 취하지 않는 `...Async`들은 `ForkJoinPool.commonPool()`(JDK 8에서 범용으로 도입된)을 사용한다. 이는 `CompletableFuture` 클래스에 있는 대부분의 메소드에도 해당된다. `runAsync()`는 간단하다. `Runnable`을 인수로 받고, 따라서 `CompletableFuture<Void>`를 반환한다. `Runnable`은 아무 것도 반환하지 않기 때문이다. 비동기 작업을 통해 결과를 받아야 하는 경우라면 `Supplier<U>`를 사용하라.

```java
final CompletableFuture<String> future = CompletableFuture.supplyAsync(new Supplier<String>() {
    @Override
    public String get() {
        //...long running...
        return "42";
    }
}, executor);
```

하지만 우리에겐 Java 8 람다가 있다!

```java
final CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
  //...long running...
  return "42";
}, executor));
```

심지어는 아래와 같이 할 수 있다.

```java
final CompletableFuture<String> future = 
    CompletableFuture.supplyAsync(() -> longRunningTask(params), executor);
```

이 글은 프로젝트 람다에 관한 것이 아니다. 하지만 광범위하게 람다를 사용할 것이다.

## CompletableFuture의 변환 (thenApply)

앞서 필자가 `CompleteableFuture`가 `Future`보다 좋다고 했는데, 아직 이유는 설명하지 않았다. 간단히 말하면, `CompleteableFuture`는 모나드<sup>monad</sup>이고 펑터<sup>functor</sup>이기 때문이다. 아마도 이 이야기가 도움이 되지 않았으리라. 스칼라와 자바스크립트는 모두 퓨처가 완료될 때 실행될 콜백을 등록할 수 있다. 준비될 때까지 기다리지 않아도 된다. 대신 이렇게 말할 수 있다. "결과가 도착하면 그 결과와 함께 이 함수를 실행시켜줘." 게다가, 이런 함수들을 스택에 쌓거나 결합하는 일 등이 가능하다. 예를 들면, `String`을 `Integer`로 변환하는 함수가 있다면, `CompletableFuture<String>`을 `CompletableFuture<Integer>`로 쉽게 변환할 수 있다. 함수를 뜯어 고치지 않고서도 말이다. 이는 `thenApply()` 등의 메소드로 가능하다.

```java
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn);
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor);
```

이전에 언급했듯이 `...Async` 버전은 `CompletableFuture`의 대부분 작업에 제공된다. 따라서, 이어지는 섹션에서의 설명은 생략한다. 단지 이것만 기억하라. 퓨처가 완료되면 첫 번째 메소드는 동일한 스레드 내에서 함수를 적용시키고, 나머지 2개 메소드는 별도의 스레드 풀에서 비동기로 적용한다.

이제 `thenApply`가 어떻게 동작하는지 살펴보자.

```java
CompletableFuture<String> f1 = //...
CompletableFuture<Integer> f2 = f1.thenApply(Integer::parseInt);
CompletableFuture<Double> f3 = f2.thenApply(r -> r * r * Math.PI);
```

혹은 아래처럼 한 문장으로도 가능하다.

```java
CompletableFuture<Double> f3 = 
    f1.thenApply(Integer::parseInt).thenApply(r -> r * r * Math.PI);
```

`String`에서 `Integer`로, 그리고 난 뒤 `Double`로 변환되는 순서가 잘 드러난다. 하지만 가장 중요한 것은, 이런 변환들이 즉각적으로 실행되거나 블럭킹되지 않는다는 점이다. 이들은 단지 기억될 뿐이고 `f1`이 완료될 때 실행된다. 일부 변환 작업의 소요 시간이 길다면, 별도의 `Executor`를 제공해서 비동기로 실행시킬 수도 있다. 이들 작업은 스칼라의 모나딕 `map`과 동등함에 주목하라.

## 완료 시 코드 실행 (thenAccept/thenRun)

계속 번역 중.
