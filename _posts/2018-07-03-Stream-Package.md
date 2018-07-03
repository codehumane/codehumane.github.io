---
layout: post
title: java.util.stream
summary: java.util.stream 패키지 설명을 읽고 요약함
date: 2018-07-03 00:00:01 +0900
---

[Associativity](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#Associativity) 속성 때문에 [Package java.util.stream Description](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)을 읽다가, 그 밖에도 재밌는 내용이 많아 요약함.

# Package java.util.stream Description

스트림 패키지를 간단히 설명하면 다음과 같음.

- 스트림 엘리먼트에 대해 함수형 스타일 연산을 지원하는 클래스들.
- 아래 코드는 스트림에 대해 filter-map-reduce 연산을 수행하는 예.

```java
int sum = widgets.stream()
                 .filter(b -> b.getColor() == RED)
                 .mapToInt(b -> b.getWeight())
                 .sum();
```

스트림이 컬렉션과 다른 점.

- No storage. 엘리먼트 저장하는 대신, 파이프라인으로 실어 나름.
- Functional in nature. 원천<sup>source</sup>을 수정하지 않고 결과를 만들어 냄.
- Laziness-seeking.
  - 많은 스트림 연산들이 최적화를 위해 lazily로 구현 가능.
  - 예컨대, `findFirst`와 같은 것은 모든 엘리먼트를 검사하지 않아도 됨.
  - 스트림 연산은 2개로 나뉨. Intermediate operation(Stream-producing)과 Terminal operation(value- or side-effect-producing).
  - Intermediate operation들은 항상 lazy.
- Possibly unbounded. 컬렉션은 유한한 크기를 가지지만 스트림은 그럴 필요 X.
- Consumable. 스트림의 엘리먼트들은 스트림의 생애 동안 단 한 번만 방문 가능. Iterator와 유사.

## Stream operations and pipelines

- 스트림 연산은 intermediate과 terminal로 나뉨.
  - Intermediate 연산들은 항상 lazy. 연산을 수행하는 대신, 새로운 스트림을 만듦.
  - Terminal 연산들은 대부분 eager. lazy 연산들이 시작되고, 결과를 만들어 냄.
  - Terminal 연산이 수행되면 스트림은 소비 되었다고<sup>Consumed</sup> 여겨지고 더 이상 사용 불가.
  - 참고로, forEach도 Terminal 연산.
- Intermediate 연산은 stateless와 stateful 연산으로 다시 나뉨.
  - stateless는 `filter`나 `map` 처럼 연산이 각 엘리먼트에 대해 독립적.
  - stateful은 결과를 만들기 전 전체 입력을 처리해야 할 수도. 스트림 정렬이 그 예.
- 일부 연산들은 short-circuiting.
  - 무한 스트림으로부터 유한 스트림을 얻는 intermediate 연산이나,
  - 무한 입력을 유한 시간 내에 처리할 수 있는 terminal 연산을 가리킴.
  - short-circuiting은 무한 스트림 연산을 위한 필요조건<sup>necessary</sup>.
  - 충분조건<sup>sufficient</sup>은 아님.

### Non-interference

- 스트림은 다양한 데이터 소스에 대해 병렬 aggregate 연산을 지원.
- 심지어 `ArrayList`와 같은 non-thread-safe 컬렉션에 대해서도 지원.
- 다만, 스트림 파이프라인이 수행되는 동안 데이터가 간섭<sup>interference</sup>되지 않을 때에만 가능.
- 여기서의 간섭이란, 스트림 파이프라인이 수행되는 동안 데이터 소스가 수정될 수 있음을 가리킴.
- 아래는 간섭의 한 예시.

```java
List<String> l = new ArrayList(Arrays.asList("one", "two"));
Stream<String> sl = l.stream();
l.add("three");
String s = sl.collect(joining(" " ));
```

### Stateless behaviors

- 스트림 연산에 대한 행위적 파라미터<sup>behavioral parameter</sup>가 stateful이면,
- 파이프라인의 결과는 비결정적<sup>nondeterministic</sup>이거나 정확하지 않을 수 있음.
- 아래는 stateful의 예시. 병렬로 연산이 수행되면 결과가 매번 달라질 수도.
- 마찬가지 이유로, 행위적 파라미터에서 가변 상태<sup>mutable state</sup>에 접근을 시도하는 것도 안정성이나 성능 측면에서 나쁜 선택이 됨.
- 상태 접근을 synchronize 하지 않으면 데이터 레이스가 발생하고 코드가 깨질 수 있음.
- 반대로, 상태 접근을 synchronize 하면 병렬화로 인해 얻을 수 있는 이점이 적어짐.

```java
Set<Integer> seen = Collections
	.synchronizedSet(new HashSet<>());

stream
	.parallel()
	.map(e -> {
		if (seen.add(e)) return 0;
		else return e;
	})...
```

### Side-effects

- 너무 당연한 이야기지만, 스트림 연산에서 행위적 파라미터에 부수효과가 있는 것은 권장되지 않음.
- statelessness 요건을 의도치 않게 위배할 수도 있고, 스레드 안전성 위험도 가질 수 있음.
- 많은 경우에 부수 효과 없는 함수들로 교체 가능함. 아래는 그 예.

```java
ArrayList<String> results = new ArrayList<>();
stream.filter(s -> pattern.matcher(s).matches())
      .forEach(s -> results.add(s)); // Unnecessary use of side-effects!

List<String> results =
    stream.filter(s -> pattern.matcher(s).matches())
          .collect(Collectors.toList()); // No side-effects!
```

### Ordering

- 스트림에서는 encounter order가 있을 수도 있고 없을 수도 있음.
- 있고 없고는 데이터 원천과 중간 연산에 달려 있음.
- 스트림 원천인 `List`나 배열인 경우에는 순서가 있고, `HashSet`과 같은 경우에는 순서 없음.
- `sorted()`와 같은 중간 연산들은 정렬되지 않은 스트림에도 순서를 부과함.
- `BaseStream.unordered()` 같은 경우에는 정렬된 스트림을 무작위 순서로 렌더링 할 수도.
- 정렬된 스트림에 대한 대부분의 연산들은 순서 제약을 가짐. 예시) [1, 2, 3]을 가지는 `List`의 경우 `map(x -> x*2)`는 [2, 4, 6]의 결과를 보장해야 함.
- 병렬 스트림에 대해서 순서 제약을 풀어주는 것이 효과적인 실행을 이끌어내기도 함.

## Reduction operations

재밌게 읽었던 부분은 바로 이 Reduction operations. 이어서 정리 예정.

