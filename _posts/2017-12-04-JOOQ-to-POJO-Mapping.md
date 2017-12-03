---
layout: post
title:  JOOQ Record의 POJO 변환
summary: JOOQ Record의 POJO 매핑 문제와 그 해결
date:   2017-12-04 00:00:01 +0900
---

# 배경

현재 맡고 있는 서비스 일부는 Type-Safe SQL을 작성하고자 [JOOQ](https://www.jooq.org/)를 사용함. 그리고 JOOQ를 통해 얻은 DB 데이터는 POJO(도메인 모델)로 변환하여 사용. JOOQ에서는 이를 위해 [org.jooq.RecordMapper](https://www.jooq.org/javadoc/3.10.x/org/jooq/RecordMapper.html)라는 것을 제공. 이는 select 문장으로 얻은 데이터에 접근할 수 있게 도와주는 콜백 API. 예를 들면, 아래와 같다.

```java
final List<Integer> ids = create
  .selectFrom(BOOK)
  .orderBy(BOOK.ID)
  .fetch()
  .map(BookRecord::getId);
```

하지만 문제가 있음. [org.jooq.Record](https://www.jooq.org/javadoc/3.6.2/org/jooq/Record.html)에 대한 POJO가 추가될 때 마다 매핑 코드가 필요한 것. 추가 뿐만이랴. 수정이 있거나 삭제가 있을 때 마다 매핑 코드는 영향을 받는다. [RecordMapperProvider](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos-with-recordmapper-provider/)라는 녀석 또한 마찬가지.

다행히, [DefaultRecordMapper](https://www.jooq.org/javadoc/3.6.1/org/jooq/impl/DefaultRecordMapper.html)를 통해 기본적인 매핑은 지원해주고 있음. 이 알고리즘은 몇 가지 관례에 의존하고 있는데, 그 관례는 [이 글](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos/)에 소개되고 있음. 아래는 몇 가지를 나열한 것.

1. [Using JPA-annotated POJOs](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos/#N48B1C)
2. [Using simple POJOs](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos/#N48B34)
3. [Using "Immutable" POJOs](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos/#N48B46)
4. [Using proxyable types](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos/#N48B58)

그리고 이것이 입맛에 맞지 않는 경우를 위해, [사용 가능한 써드 파티 3가지를 소개](https://www.jooq.org/doc/3.10/manual/sql-execution/fetching/pojos-with-recordmapper-provider/#N48BC8).

1. [ModelMapper](http://modelmapper.org/)
2. [SimpleFlatMapper](http://simpleflatmapper.org/)
3. [Orika Mapper](http://orika-mapper.github.io/orika-docs/)

# 문제

하지만, 아쉽게도 어느 한 가지 마음에 드는 게 없음. 이유를 몇 가지 나열하면 다음과 같다.

1. 외부로 노출하고 싶지 않은 필드도 무조건 노출해야 하거나(`public` 접근 제한자 또는 getter가 필요),
2. 변환 대상이 늘어나거나 변경되거나 추가될 때 마다 매핑 코드를 함께 수정해 주어야 하거나,
3. `Enum` 타입에 대해 기초적인 수준만 지원하거나,
4. 심각한 버그가 있거나(이건 ModelMapper 이야기),
5. 필드명을 대응시키는 알고리즘이 너무 관대하거나, 혹은 엄격한 경우 우리 상황에 맞지 않거나,
6. 반복적인 코드를 매번 작성해주어야 하거나,
7. 부족하다면 확장할 수 있어야 하는데, 문서는 커녕 내부 코드를 이곳 저곳 뒤져봐도 그런 게 없거나.

문제가 없고 아쉬움이 없는 오픈소스가 어디 있으랴(물론 있다). 좋은 대안을 찾아보거나, 문제를 우회해서 사용하거나, 구조적인 변경 등을 통해 어떻게든 사용하곤 함. 문제를 해결하는 방법은 다양하니까. 그리고 개발자가 조금 불편하면 어떤가. 사용자가 좋으면 되지.

하지만, 레거시 시스템을 신규로 구축해가는 과정이다 보니 극복해야 하는 DB 상의 제약이 많았고, 반복적으로 작성해야 하는 코드도 많았음. 이를 꼭 해결하고 싶었음. 레거시의 한계를 극복하려고 신규 시스템을 구축하는 거였으니까. 문서도 뒤져보고, 문서로는 부족해서 내부 코드도 살폈으나(그 과정에서 버그들과 확장 불가한 구조들을 마주침), 해결방법이 마땅치 않았다. 물론, 역량 부족으로 찾지 못했을 수도 :)

# 해결

그래서 이틀 정도 투자하여 Mapper를 직접 작성함. 위 6가지 문제를 모두 극복(단, 7번은 내부 코드라서 고려 안함). 로직은 아래처럼 간단하다.

1. POJO의 필드와 `Record`의 필드를 모두 추출.
2. 정규식을 통해 각 필드를 토크나이징<sup>tokenize</sup>하고, 모든 구분 단어가 일치하는지를 대소문자 무시하고 비교.
3. 토크나이징을 한 이유는 POJO 필드명은 CamelCase이고 `Record` 필드명은 snake_case 형태이기 때문.
4. 최종적으로 타입 비교하고, 타입 불일치할 경우 런타임 에러 발생.
5. 필드명과 타입이 일치하면 POJO의 필드에 값을 할당.
6. 그 외 서비스에서 정의한 특수 타입(확장된 Enum 등)에 대한 처리 추가.

토크나이저 코드만 단순화하여 소개하면 다음과 같음.

```java
class JooqFieldTokenMatcher {

  static boolean match(org.jooq.Field<?> jooqField, Field pojoField) {
    final String[] pojoTokens = camelCaseTokenizer.tokenize(pojoField.getName());
    final String[] jooqTokens = snakeCaseTokenizer.tokenize(jooqField.getName());

    if (jooqTokens.length != pojoTokens.length)
      return false;
    
    for (int i = 0; i < jooqTokens.length; i++) {
      if (!jooqTokens[i].equalsIgnoreCase(pojoTokens[i]))
        return false;
    }
    
    return true;
  }

  static class NumberIgnoreCamelCaseTokenizer {
    
    private static final String UPPER_AND_UPPER_LOWER = "(?<=[A-Z])(?=[A-Z][a-z0-9])";
    private static final String NON_UPPER_AND_UPPER = "(?<=[^A-Z])(?=[A-Z])";
    private static final String ETC = "(?<=[A-Za-z0-9])(?=[^A-Za-z0-9])";
    private final Pattern camelCase = Pattern
      .compile(String.format(
        "%s|%s|%s",
        UPPER_AND_UPPER_LOWER,
        NON_UPPER_AND_UPPER,
        ETC
      ));

    String[] tokenize(String name) {
      return camelCase.split(name);
    }
  }

  static class SnakeCaseTokenizer {
    
    private final Pattern underscore = Pattern.compile("_");
    
    String[] tokenize(String name) {
      return underscore.split(name);
    }
  }
}
```

*반대 방향의 변환(POJO -> JOOQ)도 비슷한 방식으로 구현.

# 부하

하지만 부하 테스트를 할 경우 성능 저하를 일으킴. 그래서 아래와 같이 성능 개선.

1. 부하의 원인으로 3가지를 추정. 하나씩 실험해 보기로 함.
   - [Stream.forEach](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#forEach-java.util.function.Consumer-)의 많은 사용. 이유는 [여기](https://blog.jooq.org/2015/12/08/3-reasons-why-you-shouldnt-replace-your-for-loops-by-stream-foreach/)를 참고.
   - 필드 토크나이징 비교를 할 때 수행되는 중첩 루프. O(n^2) 시간복잡도.
   - 많은 [리플렉션](https://docs.oracle.com/javase/tutorial/reflect/) 연산. 이유는 [여기](https://docs.oracle.com/javase/tutorial/reflect/)의 **Performance Overhead** 부분 참고.
2. 첫 번째를 for-loop으로 교체해봤지만 별다른 성과 없음.
3. 2번과 3번을 줄이기 위해 캐시 도입.
   - Record -> POJO 변환 1회 시도
   - 해당 타입의 Record, POJO를 키로 하여 매핑 연산 캐시
   - 동일한 타입의 Record와 POJO를 변환하려는 경우 캐시된 연산 재사용
4. 3번을 시도한 결과 TPS가 2배 좋아짐.
5. 추가적인 개선 포인트가 보였으나, 보수적으로 접근하기로 하고, 더 이상 진행하지 않음.

# 배포

성능까지 개선했으나, 개선된 부분은 애플리케이션의 핵심 부분. 즉, 많은 곳에서 사용중. 그래서 아래와 같이 고민.

1. 일단 영향 범위 파악.
   - 개선된 함수가 여전히 설계 계약<sup>Design Contract</sup>을 지키고 있었고,
   - 기존 코드의 큰 변화 없이 캐싱 로직만 추가되었을 뿐이며,
   - 따라서 캐싱 로직에 결함이 있지 않는 이상 영향 범위는 없어보였음.
   - 또한, 캐싱 로직도 너무나 단순.
2. 그럼에도 불구하고 보수적으로 접근하기로 함.
   - 우선 하루 정도의 QA를 거치고,
   - [Feature Toggle](https://martinfowler.com/articles/feature-toggles.html)을 적용하여 언제든 새로운 로직을 기존 로직으로 대체할 수 있도록 하고,
   - [Canary Release](https://martinfowler.com/bliki/CanaryRelease.html) 개념을 차용해서, 새로운 로직이 점진적으로 사용될 수 있도록 함.