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

```java
CompletableFuture<Void> thenAccept(Consumer<? super T> block);
CompletableFuture<Void> thenRun(Runnable action);
```

이 두 메소드는 퓨처 파이프라인에서의 전형적인 "마지막" 단계이다. 이들은 퓨처 결과가 구해지면, 소비할 수 있게 도와준다. `thenAccept()`는 그 마지막 값을 제공하지만, `thenRun`은 `Runnable`을 실행, 즉 연산 결과에 접근이 불가하다.

```java
future.thenAcceptAsync(db1 -> log.debug("Result: {}", db1), executor);
log.debug("Continuing");
```

`…Async`의 변형들은 두 메소드에 대해서도 제공된다. 또한, `executor`를 인자로 받는 것도 있고 받지 않는 것도 있다. 여기서 중요한 것은 `thenAccept()/thenRun()` 메소드가 블럭킹을 일으키지 않는다는 점이다(명시적으로 `executor`가 없다고 하더라도 말이다). 이들을 마치 퓨처에 할당된 이벤트 리스너/핸들러처럼 여기길 바란다. 그리고 곧 실행될 것이다. 물론, "Continuing" 메시지가 먼저, 그리고 즉각적으로 나타난다. `future`가 아무리 빨리 완료된다고 하더라도 말이다.

## 단일 CompletableFuture의 에러 핸들링

지금까지 우리는 연산 결과에 대해서만 살펴보았다. 예외는 어떻게 해야 할까? 이들 역시 비동기로 다룰 수 있을까? 물론이다!

```java
CompletableFuture<String> safe =
  future.exeptionally(ex -> "We have a problem: " + ex.getMessage());
```

`exceptionally()`에서 인자로 취한 함수는 `future`가 예외를 던질 때 실행된다. 여기서 이 예외를 다시 `Future`의 타입과 호환되는 값으로 변환할 수 있다. `safe`의 이어지는 변환에는 예외가 전파되는 대신 함수가 반환하는 문자열을 받게 된다.

좀 더 유연한 접근법은 `handle()`을 사용하는 것이다. 이는 정상적인 결과와 예외 모두를 인자로 받는다.

```java
CompletableFuture<Integer> safe = future.handle((ok, ex) -> {
  if (ok != null) {
    return Integer.parseInt(ok);
  } else {
    log.warn("Problem", ex);
    return -1;
  }
});
```

`handle()`은 항상 호출되며, 정상적인 결과나 예외 인자 둘 중 하나는 널값이 아니다. 결국, 한 번에 캐치하는 전략이다.

## 두 개의 CompletableFuture 조합

한 개의 `CompletableFuture`를 비동기로 처리하는 것이 멋지긴 하지만, 다수의 퓨처가 다양한 방식으로 조합될 때 더 큰 힘을 발휘한다.

### 두 개의 퓨처 조합/체이닝 (thenCompose())

퓨처의 값에 대해 실행되지만, 반환되는 값이 퓨처인 경우가 있다. `CompletableFuture`는 충분히 똑똑해서, 함수의 결과를 `CompletableFuture<CompletableFuture<T>>`로 반환하는 대신, 최상위 레벨의 퓨처를 반환해 줄 수 있다. 그런 측면에서 `thenCompose()` 메소드는 스칼라에서의 `flatMap`과 동등하다.

```java
<U> CompletableFuture<U> thenCompose(Function<? super T, CompletableFuture<U>> fn);
```

아래의 예제에서는 `calculateRelevance()` 함수가 적용되는 부분에서, `thenApply()(map)`과 `thenCompose()(flatMap)`의 차이, 그리고 타입들을 유심히 살펴보라. 물론 `thenCompose`에도 `…Async` 변형이 존재한다.

```java
CompletableFuture<Document> docFuture = //...
CompletableFuture<CompletableFuture<Double>> f =
  docFuture.thenApply(this::claculateRelevance);
CompletableFuture<Double> relevanceFuture =
  docFuture.thenCompose(this::calculateRelevance);

private CompletableFuture<Double> calculateRelevance(Document doc) // ...
```

`thenCompose()`는 튼튼한 비동기 파이프라인을 만들 때 필수적인 메소드이다. 중간 단계들을 기다리거나 블럭킹 시키지 않으면서 말이다.

### 두 개의 퓨처 변환 (thenCombine())

`thenCompose`가 하나의 퓨처를 다른 의존 퓨처에 체이닝하는데 사용되는 한편, `thenCombine`은 두 개의 독립적인 퓨처를 결합(두 개 모두 완료되면)해준다.

```java
<U,V> CompletableFuture<V> thenCombine(CompletableFuture<? extends U other, BiFunction<? super T,? super U,? extends V> fn);
```

두 개의 `CompletableFuture`가 있다고 상상해보자. 하나는 `Customer`를 불러오고 다른 하나는 가까운 `Shop` 정보를 불러온다. 이들은 서로 독립적으로 완료되지만, 둘 모두가 완료되면, 이들의 값을 `Route` 계산하는데 활용하고 싶다. 여기 그 예제가 있다.

```java
CompletableFuture<Customer> customerFuture = loadCustomerDetails(123);
CompletableFuture<Shop> shopFuture = closestShop();
CompletableFuture<Route> routeFuture = 
  customerFuture.thenCombine(shopFuture, (cust, shop) -> findRoute(cust, shop));

//...
 
private Route findRoute(Customer customer, Shop shop) //...
```

Java 8에서는 `(cust, shop) -> findRoute(cust, shop)`을 `this::findRoute` 메소드 레퍼런스로 단순화할 수 있다.

```java
customerFuture.thenCombine(shopFuture, this::findRoute);
```

### 두 개의 CompletableFuture가 모두 완료되기를 기다리기

새로운 `CompletableFuture`를 만들어 두 개의 결과를 조합하는 대신, 작업이 완료되면 통지받는 간단한 방법도 있다. 이 경우 `thenAcceptBoth()/runAfterBot()` 등의 메소드를 사용한다. `thenAccept()`와 `thenRun()`과 유사하지만 한 개 대신 두 개의 퓨처를 기다린다.

```java
<U> CompletableFuture<Void> thenAcceptBoth(CompletableFuture<? extends U> other, BiConsumer<? super T,? super U> block);
CompletableFuture<Void> runAfterBoth(CompletableFuture<?> other, Runnable action);
```

앞선 예제에서 처럼, 새로운 `CompletableFuture<Route>`를 만드는 대신, 즉각적으로 이벤트 전송이나 GUI 리프레시 등을 하고 싶을 수 있다. `thenAcceptBoth`를 이용하면 쉽다.

```java
customerFuture.thenAcceptBoth(shopFuture, (cust, shop) -> {
  final Route route = findRoute(cust, shop);
  //refresh GUI with route
});
```

이렇게 묻는 사람들이 있을지도 모르겠다. "두 퓨처를 간단히 블럭시키면 되지 않을까?" 아래처럼 말이다.

```java
Future<Customer> customerFuture = loadCustomerDetails(123);
Future<Shop> shopFuture = closestShop();
findRoute(customerFuture.get(), shopFuture.get());
```

물론 가능한 일이다. 하지만 `CompletableFuture`를 사용하는 것은 비동기적으로, 이벤트 주도 프로그래밍 모델을 사용하기 위함이다. 블럭킹되거나 성급하게 결과를 기다리지 않고 말이다. 따라서, 위 2개의 코드 블럭은 동등하긴 하나, 후자의 것은 불필요하게 스레드의 실행을 점유해 버린다.

## 먼저 완료되는 CompletableFuture 기다리기

`CompletableFuture`는 다수의 퓨처 중 가장 먼저 완료된 것의 결과를 받을 수도 있다. 결과 타입이 동일한 두 개의 작업이 있고, 어떤 작업이 먼저 끝나는지 보다 응답 시간이 중요할 때, 이 기능이 유용할 수 있다.

```java
CompletableFuture<Void> acceptEither(CompletableFuture<? extends T> other, Consumer<? super T> block);
CompletableFuture<Void> runAfterEither(CompletableFuture<?> other, Runnable action);
```

두 개의 시스템을 통합해야 한다고 가정해보자. 하나는 빠른 평균 응답 시간을 가지지만 들쭉 날쭉하고, 다른 하나는 느리지만 일관된 응답 시간을 가진다. 이 때 취할 수 있는 한 가지 방법이 있다. 둘을 동시에 호출한 뒤 하나가 완료되길 기다리는 것이다. 일반적으로는 첫 번째가 그 대상이 되고, 다소 느려지더라도 두 번째 작업에 의해 예측 가능한 수준에서 완료된다.

```java
CompletableFuture<String> fast = fetchFast();
CompletableFuture<String> predictable = fetchPredictably();
fast.acceptEither(predictable, s -> {
  System.out.println("Result: " + s);
});
```

여기서 `s`는 `fetchFast()` 혹은 `fetchPredictably()`의 결과 문자열이다. 그게 누군지는 알 수도 없고 신경쓰지도 않는다.

### 먼저 완료된 결과 변환

`applyToEither()`는 `acceptEither()`의 형이다. 후자는 퓨처의 결과를 소비하는 반면, `applyToEither()`는 새로운 퓨처를 반환한다. 그리고 마찬가지로, 두 개의 퓨처 중 하나라도 완료되면 함께 완료된다.

```java
<U> CompletableFuture<U> applyToEither(CompletableFuture<? extends T> other, Function<? super T,U> fn);
```

`fn` 함수는 먼저 완료된 퓨처의 결과에 대해 실행된다. 필자 개인적으로는 이 특수화된 메소드의 목적을 모르겠다. 다음과 같이 쉽게 사용할 수 있는 방법이 있는데도 말이다.

```java
fast.applyToEither(predictable).thenApply(fn);
```

API에 명시되어 있긴 하지만, 실제로 필요로 하지는 않으므로, 단순히 `Function.identity()`라는 표시자를 사용했다.

```java
CompletableFuture<String> fast = fetchFast();
CompletableFuture<String> predictable = fetchPredictably();
CompletableFuture<String> firstDone =
  fast.applyToEither(predictable, Function.<String>identity());
```

`firstDone` 퓨처는 이제 다른 사용자에게 전달되고, 사용자는 두 개의 퓨처가 숨겨져 있다는 사실을 모른다. 단지 퓨처가 완료되길 기다릴 뿐이다. `applyToEither()`만이 먼저 끝나는 퓨처로부터 결과를 전달 받는다.

## 다수의 CompletableFuture 조합

두 개의 퓨처가 완료되길 기다리는 법(`thenCombine()`), 그리고 둘 중에 먼저 완료되는 퓨처만을 기다리는 법(`applyToEither()`)도 알았다. 하지만 더 많은 퓨처에 대해서는 어떻게 해야 할까? 이 때 아래의 `static` 헬퍼 메소드를 사용할 수 있다.

```java
static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs);
static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs);
```

`allOf()`는 퓨처를 배열로 받고 한 개의 퓨처를 반환한다. 그리고 배열로 받은 모든 퓨처가 완료될 때, 반환된 퓨처도 완료된다. 한편, `anyOf()`는 배열로 받은 퓨처 중 하나라도 먼저 완료되면, 반환된 퓨처를 완료시킨다. 반환된 퓨처의 제너릭 타입에 주목하라. 아마도 당신이 기대한 것은 아니었을 것이다. 다음 글에서 이 이슈에 대해 다룰 것이다.

## 번역을 마치며

원문서가 워낙 설명을 잘해줘서 종종 잊곤 했지만, `CompletableFuture`가 그리 직관적인 API를 제공하고 있다고 생각되지 않는다. 다른 비동기 API들이 상대적으로 익히고 사용하기에 쉬웠기 때문이다. 물론, 역량 부족일 수도 있고, 제대로 API를 살펴보지 않아서 그럴 수도 있다. 그 외에도 얼마든지 이유가 있겠지만, 어쨌든 당분간은 약간의 의구심과 함께 익히고 사용하려 한다. 자, 여기까지가 끝이다. 참으로 어색한 급 마무리지만, 공부가 목적인 번역이었으니, 이대로 그냥 끝맺음을 :)

