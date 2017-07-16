---
layout:     post
title:      OptionalDataException, 동시성 이슈 해결하기
date:       2017-07-15 12:32:18
summary:    Apache Shiro의 내부 동작을 이해하고 테스트하며, 동시성 문제를 해결해 나가는 과정을 기술함
---

# 문제 발견

Java, Spring, Tomcat, Apache Shiro, EhCache 등의 도구를 사용하는 웹 애플리케이션에서, 사용량이 많은 경우 아래와 같이 API가 간헐적으로 500을 응답함.

![access-log-500]({{ site.baseurl }}/images/optional-data-exception/access-log-500.png)

이 경우 사용자들은 로그인 실패, 오류 페이지 이동 등의 문제를 겪음. 수 차례 재현해 보려 했으나 실패. 문제가 발생한 사용자도 재시도하면 바로 성공함. 뭐가 문제일까?

# 문제 파악하기 1. 로그 확인

JVM, Tomcat 스레드 덤프, 각종 캐시 지표, DB Slow Query, OS 메모리, 디스크, 네트워크 등 확인. 커밋 이력, 네트웍 작업 등 최근의 변경 사항 확인. 하지만, 특별히 의심할 만한 부분 없음. 간헐적으로 발생하는 오류이므로, 당장 어떤 조치를 취하기보다 시간을 두고 원인 찾아보기로 결정함.

먼저, 문제 발생 시점의 Catalina(Tomcat's Servlet Container) 로그부터 다시 살펴봄.

```
심각: Servlet.service() for servlet [apiServlet] in context with path [] threw exception [org.apache.shiro.cache.CacheException: net.sf.ehcache.CacheException: Uncaught exception in get() - java.io.OptionalDataException] with root cause
java.io.OptionalDataException
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1370)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:370)
	at java.util.HashMap.readObject(HashMap.java:1179)
	at sun.reflect.GeneratedMethodAccessor119.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at java.io.ObjectStreamClass.invokeReadObject(ObjectStreamClass.java:1017)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1893)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1798)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1350)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:370)
	at org.apache.shiro.session.mgt.SimpleSession.readObject(SimpleSession.java:500)
	at sun.reflect.GeneratedMethodAccessor113.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at java.io.ObjectStreamClass.invokeReadObject(ObjectStreamClass.java:1017)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1893)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1798)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1350)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:370)
	at net.sf.ehcache.SerializationModeElementData.create(SerializationModeElementData.java:28)
```

눈에 띄는 부분은 2가지.

1. `java.io.OptionalDataException`
2. `org.apache.shiro.session.mgt.SimpleSession.readObject(SimpleSession.java:500)`

문제를 해결하려면 `OptionalDataException`에 대한 이해가 필요하다고 판단. 이 녀석의 정체를 살펴보기 시작함.

# 문제 파악하기 2. `OptionalDataException` 이해

`OptionalDataException`(이하 `ODE`)이 도대체 뭐지? [오라클 Java API 문서](https://docs.oracle.com/javase/7/docs/api/java/io/OptionalDataException.html)의 설명에 따르면, 이 예외는 다음의 2가지 경우에 발생.

> 1. An attempt was made to read an object when the next element in the stream is primitive data. In this case, the OptionalDataException's length field is set to the number of bytes of primitive data immediately readable from the stream, and the eof field is set to false.
> 2. An attempt was made to read past the end of data consumable by a class-defined readObject or readExternal method. In this case, the OptionalDataException's eof field is set to true, and the length field is set to 0.

첫 번째는 스트림에서 읽으려고 하는 다음 엘리먼트가 원시 데이터<sup>primitive type</sup>인데, 객체로 읽으려는 시도가 있는 경우를 가리킴. 그런데 잠깐. 내가 이해한 게 정말 맞는 걸까? 확인을 위해 간단히 코드를 작성함.

```java
@Test
void serialize_다음_읽기순서가_원시타입인데_객체로_읽기를_시도하면_ODE_예외가_발생한다() throws Exception {

  // given
  val write1st = new Person("al-Khwārizmī", 1237);
  val write2nd = new Person("Fibonacci", 847);
  val write3rd = 3;

  // given
  val oos = new ObjectOutputStream(new FileOutputStream(file));
  oos.writeObject(write1st);
  oos.writeObject(write2nd);
  oos.writeInt(write3rd);
  oos.close();

  // given
  val ois = new ObjectInputStream(new FileInputStream(file));
  ois.readObject();
  ois.readObject();

  // when
  assertThrows(OptionalDataException.class, () -> {
    // 원시 타입인데 객체로 읽으려고 시도
    ois.readObject();
  });
}
```

기대한 대로 동작함. 스트림의 3번째 데이터는 `writeInt`로 작성된 원시 타입인데, `readObject`로 객체를 읽으려 하니 `ODE` 발생. 만약, `readInt`로 읽어들이면 잘 동작함. 전체 코드는 [여기](https://github.com/codehumane/troubleshoot-java/blob/master/optional-data-exception/src/OptionalDataExceptionTest.java)를 참고.

두 번째는, 클래스에 정의된 `readObject`나 `readExternal` 메소드에서 더 이상 읽어들일 데이터가 없는데도 읽기를 시도하는 경우를 가리킴. 여기서 "클래스에 정의된 `readObject`나 `readExternal`"에 대한 설명은 [오라클 Java API 문서의 Serializable](https://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html)을 참고. 자, 이번에도 내가 이해한 게 맞는 걸까? 확인을 위해 마찬가지로 간단한 코드를 작성함.

```java
@Test
void serialize_readObject_직접_구현시_EOF를_지난_읽기를_시도하면_ODE_예외가_발생한다() throws Exception {

  // given
  val person = new PersonWithSerializeHandling("al-Khwārizmī", 1237);

  // given
  val oos = new ObjectOutputStream(new FileOutputStream(file));
  oos.writeObject(person);
  oos.close();

  // given
  val ois = new ObjectInputStream(new FileInputStream(file));

  // when
  assertThrows(OptionalDataException.class, () -> {
    // `readObject` 직접 구현시, EOF 지난 읽기를 시도
    ois.readObject();
  });
}

class PersonWithSerializeHandling implements Serializable {

  private String name;
  private Integer age;

  PersonWithSerializeHandling(String name, Integer age) {
    this.name = name;
    this.age = age;
  }

  private void writeObject(ObjectOutputStream out)
      throws IOException {
    
    out.writeObject(name);
    // 고의로 누락시킨 writeObject
    // out.writeObject(age);
  }

  private void readObject(ObjectInputStream in)
      throws IOException, ClassNotFoundException {
    name = String.class.cast(in.readObject());
    age = Integer.class.cast(in.readObject());
  }
}
```

`writeObject`는 한 번만 했는데, `readObject`는 2번 시도하니 `ODE` 발생함. 참고로, 이 작업을 클래스에 정의된 `readObject`가 아닌, 다른 곳에서 수행하면 `ODE`가 아니라 `EOF` 예외가 발생함. 전체 코드는 [여기](https://github.com/codehumane/troubleshoot-java/blob/master/optional-data-exception/src/OptionalDataExceptionTest.java)를 참고. 이제 어느 정도 이해가 되었다고 판단. 다음 단계로 넘어감.

# 문제 파악하기 3. `SimpleSession` 살펴보기

이제 에러 로그에서 2번째로 눈에 띄던 부분을 살펴보기로 함.

> `org.apache.shiro.session.mgt.SimpleSession.readObject(SimpleSession.java:500)`

문제가 되는 지점의 코드는 아래와 같음. 참고로, 당시 사용중인 `Apache Shiro`의 버전은 `1.2.3`. 전체 코드는 [여기](https://shiro.apache.org/static/1.2.3/apidocs/src-html/org/apache/shiro/session/mgt/SimpleSession.html)를 참고.

```java
@SuppressWarnings({"unchecked"})
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
  in.defaultReadObject();
  short bitMask = in.readShort();
  if (isFieldPresent(bitMask, ID_BIT_MASK)) {
    this.id = (Serializable) in.readObject();
  }
  if (isFieldPresent(bitMask, START_TIMESTAMP_BIT_MASK)) {
    this.startTimestamp = (Date) in.readObject();
  }
  if (isFieldPresent(bitMask, STOP_TIMESTAMP_BIT_MASK)) {
    this.stopTimestamp = (Date) in.readObject();
  }
  if (isFieldPresent(bitMask, LAST_ACCESS_TIME_BIT_MASK)) {
    this.lastAccessTime = (Date) in.readObject();
  }
  if (isFieldPresent(bitMask, TIMEOUT_BIT_MASK)) {
    this.timeout = in.readLong();
  }
  if (isFieldPresent(bitMask, EXPIRED_BIT_MASK)) {
    this.expired = in.readBoolean();
  }
  if (isFieldPresent(bitMask, HOST_BIT_MASK)) {
    this.host = in.readUTF();
  }
  if (isFieldPresent(bitMask, ATTRIBUTES_BIT_MASK)) {
    // 에러 발생 지점! 여기에요 여기!
    this.attributes = (Map<Object, Object>) in.readObject();
  }
}
```

일단, 코드를 보고 간단히 확인할 수 있었던 내용은 다음과 같음.

1. `SimpleSession`은 `Serializable` 구현체.
2. `readObject`, `writeObject`를 직접 구현하고 있음.
3. 또 한 가지는 원시 타입이 serialize/deserialize 대상에 포함되지 않았다는 것.
4. 모든 필드가 `tansient`로 선언되어 있고, `readObject`, `writeObject`에서도 원시 타입을 다루지 않기 때문임.
5. 그 외에도 비트 마스크를 이용하여, 스트림에서 읽고 쓰는 데이터를 동적으로 결정하는 부분이 흥미로움.

위 사실을 통해 `ODE`의 2가지 발생 원인 중, 원시 타입에 관련된 것은 아님을 알 수 있음. 그렇다면 읽어 들일 데이터가 없는데 읽기를 시도했다는 이야기. 예외가 발생한 지점의 코드를 다시 살펴봄.

`this.attributes = (Map<Object, Object>) in.readObject();`

혹시나 싶어 `writeObject` 메소드 내부를 살펴봤으나, `this.attributes`에 대한 write를 순서에 맞게 잘 수행하고 있음. write를 제대로 했는데, 왜 read를 못하는 걸까? `if` 조건절의 비트 마스크 처리 로직이 잘못되었나 싶었지만, `isFieldPresent` 메소드 내부를 살펴보면 의심은 금방 사라짐. 별다른 게 없음. 매우 간단함. 도대체 뭐가 문제인가?

# 문제 파악하기 4. read 동작 이해

로그와 코드만으로는 잘 파악이 안되서, 조금 다른 질문으로 접근을 시도.

1. 이 코드(read이자 deserialize)는 언제 호출되나?
2. read 되는 데이터는 무엇인가?

## 언제 호출되나?

디버깅을 하며 확인한 내용은 아래와 같음.

1. 문제가 발생하는 지점은 `SimpleSession`의 `readObject`이고,
2. EhCache가 캐시 저장소로부터 데이터를 읽어 들이려고(deserialize) 할 때 발생하며,
3. 이는 요청 스레드가 시작될 때, Apache Shiro가 자신의 Servlet Filter에서 세션 데이터를 가져오려고 하는 시점임.

## 무슨 데이터인가?

디버깅을 통해 어떤 데이터가 `this.attribute`에 담기는지 살펴봄. 이 부분에서 살짝 놀랐고, 문제 해결 실마리가 보이기 시작함.

1. `SimpleSession`은 Apache Shiro가 인증을 위해 자체적으로 필요한 데이터를 담는 곳
2. 이 중에서 `attribute` 필드에는 애플리케이션에서 관리하고 싶은 데이터를 담을 수 있음.
3. 그런데 이 필드에 담긴 데이터가 상당히 많음. 대략 30여개 항목.
4. 참고로, 직접 `SimpleSession`에 데이터를 담는 것은 아니고, 2가지 추상화 된 인터페이스를 통해 담긴 것.

데이터가 많은 것을 보고 이런 의심이 들었음.

> 혹시 write가 아직 끝나지 않았는데, read를 시도하려는 것 아닐까?

# 문제 파악하기 5. write 동작 이해

"write가 아직 끝나지 않았는데, read를 시도하려는 것 아닐까?"라는 가설을 검증하기 위해, write 동작을 좀 더 살펴보기 시작함. 살펴 본 내용은 다음과 같음.

1. HTTP 요청이 들어올 때 마다, `SimpleSession` 업데이트가 발생함.
2. 이는 Apache Shiro의 [Session Timeout](https://shiro.apache.org/session-management.html#session-timeout) 기능 때문임.
3. 매 HTTP 요청마다 세션 데이터에 대한 접근이 발생하고,
4. 이 때 마다 `SimpleSession`의 `lastAccessTime` 필드가 현재시간으로 갱신됨.
5. 그리고 이를 저장소(이 경우에는 캐시 서버)에 반영하기 위해 write(serialization) 발생함.
6. 자세한 내용은 [`AbstractShiroFilter`의 `updateSessionLastAccessTime`](https://shiro.apache.org/static/1.2.3/apidocs/src-html/org/apache/shiro/web/servlet/AbstractShiroFilter.html#line.307)을 참고.

그런데, `SimpleSession`은 당연히(?) 세션별로 하나씩 존재함. 동일 `SimpleSession`에 대해 write와 read가 동시에 일어나려면, 동일한 세션의 HTTP 요청이 동시에 발생해야 함. 이 당시의 애플리케이션은 하나의 화면을 구성하기 위해, 여러 개의 Ajax 요청이 발생하는 구조였음. 즉, 동일한 세션의 HTTP 요청이 동시에 발생 가능함. 그림으로 보면 아래와 같음.

![ODE-on-read]({{ site.baseurl }}/images/optional-data-exception/ode-on-concurrent-write-read.jpg)

# 해결 방법 모색 1. 동시성 조건 제거

마틴 파울러의 [엔터프라이즈 애플리케이션 아키텍처 패턴](http://wikibook.co.kr/peaa/)에 보면, 동시성 문제가 발생하는 조건을 다음과 같이 이야기함.

> 둘 이상의 사용자가 동일한 데이터를 사용하려 하는 경우

세부적으로는 이것이 2가지 조건의 논리곱이라고 봤음. 따라서, 이 중 하나만 거짓을 만들면 동시성 발생 안함.

1. 대상 데이터에 대해 한 번에 한 사용자만 접근을 허용
2. 데이터를 읽기 전용으로 만듦

전자는 격리<sup>isolation</sup>, 후자는 불변성<sup>immutable</sup>.

`SimpleSession`에 대해서도 마찬가지라고 생각하여, 첫 번째로 격리를 생각함. [BFF<sup>Backend For Frontend</sup>](http://samnewman.io/patterns/architectural/bff/) 등을 활용하여, 한 화면에서 여러 개로 나누어 보내던 요청을 하나로 합치는 것. 어느 정도만 합쳐도 `ODE` 발생 가능성은 꽤 낮아질 것. 그러나 단기간에 그 많은 화면들을 작업하기에는 부담.

두 번째로, 불변성을 생각함. `lastAccessTime`을 업데이트 하지 않는다거나(write 없애기), 10번에 1번만 업데이트를 하는 것(write 횟수 줄이기). 그러나 Apache Shiro는 무조건 `lastAccessTime`을 업데이트 하고 있음. Shiro에서 설정이나 인터페이스를 제공하는지 문서와 코드를 살펴봤지만 찾지 못함. 결국 내부 코드를 건드려야 함. 10번에 1번만 업데이트 하는 것도 마찬가지.

# 해결 방법 모색 2. 동시성 대상 제거

> 그냥 동시성 문제를 일으키는 데이터 자체를 없애는 것은 어떨까?

격리나 불변에 관련된 행위를 바꾸기는 어려움. 그래서 그냥 문제가 되는 데이터 자체를 없애버리기로 함. 다행스럽게도 에러 발생지점을 살펴보면, `SimpleSession`의 모든 필드가 아닌 `attributes`에 대해서만 동시성 문제가 발생. 이 부분은 Shiro 데이터가 아닌 애플리케이션에서 필요로 하는 데이터. 반드시 `SimpleSession`을 통해 관리할 필요는 없음.

따라서, 애플리케이션에서 관리하는 세션 데이터를 `SimpleSession`으로부터 분리하기로 결정.

# 구현. 세션 데이터 분리하기

다행히 `SimpleSession`의 `attributes` 필드에 저장되는 데이터는 2가지 인터페이스를 통해 이뤄지고 있었음. 이 2개의 구현체를 새로 만들어 교체하면 됨. 인터페이스의 입력값은 세션 아이디, 반환값은 `attributes` 데이터를 적절히 변환한 값객체. 구현 내용을 간단히 기록하면 다음과 같음.

1. 캐시 서버에 별도의 region을 마련. 애플리케이션의 세션 데이터가 담길 곳임.
2. 데이터의 키는 Apach Shiro의 `sessionId`, 값은 `SimpleSession`의 `attributes`에 담기던 데이터.
3. 새로 분리된 데이터는 `SimpleSession`과 1:1 대응됨.
4. 새로 분리된 데이터는 `SimpleSession`과 생명주기를 같이함. 이를 위해 [`SessionListenerAdapter`](http://shiro.apache.org/static/1.3.2/apidocs/org/apache/shiro/session/SessionListenerAdapter.html) 등이 활용됨.
5. 속도 향상을 위해, [ThreadLocal](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html) 활용. 요청의 시작과 끝 사이에 스레드가 캐시 서버에서 한 번이라도 세션 데이터를 읽어들이면, 이를 스레드의 로컬 영역에 저장함.
6. 스레드는 풀 방식으로 관리되므로, 요청이 완료될 때 `ThreadLocal` 비워줌.

간단히 그림으로 표현하면 아래와 같음 (세션 데이터 write, 만료나 중단에 따른 remove는 생략. read에 대해서만 표현함)

![ODE-on-read]({{ site.baseurl }}/images/optional-data-exception/extract-simple-session.jpg)

코드 배포 이후로는 `ODE` 발생하지 않음. 전체 코드는 [여기](https://github.com/codehumane/cache-handling/tree/master/src/main/java/session)에 기록함.

# 버그픽스. `ThreadLocal` 갱신 조건

`ODE`는 더 이상 발생하지 않았으나, 이따금 버그가 발생함. `ThreadLocal`로 데이터를 관리하는 방식이 다음과 같았기 때문.

1. 애플리케이션 세션 데이터 읽기가 발생하면,
2. `ThreadLocal`에 데이터가 존재하는지, 그리고 동일한 세션 아이디를 가진 값인지 판단함.
3. 두 조건 모두 만족하면 `ThreadLocal` 데이터를 사용하고,
4. 그렇지 않으면 캐시 서버로부터 데이터를 가져옴.
5. 그리고 `ThreadLocal` 데이터를 새로 가져온 데이터로 교체함.

문제가 되는 부분은 2번의 판단 조건. 대부분의 경우 문제가 없었으나, 딱 한가지 문제되는 경우가 존재.

> 스레드가 두 번 연속 동일한 세션 요청에 할당 되었는데, 이 두 개의 요청 사이에 다른 요청 스레드에서 해당 세션이 업데이트 됨.

그림으로 보면 다음과 같음.

![threadlocal-reset-bug]({{ site.baseurl }}/images/optional-data-exception/threadlocal-reset-bug.jpg)

필터를 등록해서 스레드가 반납될 때 마다, 혹은 스레드가 사용되기 시작할 때마다 `ThreadLocal`을 비웠다면 문제가 되지 않았을 것. 처음부터 그렇게 하지 않은 이유는 필터 등록이 번거롭게 느껴졌기 때문임. 가능한 필터 등록을 피하고 싶었음. 하지만 다른 방법을 찾지 못하고 결국 필터를 등록하여 문제를 해결함. 이 변경 작업은 [여기](https://github.com/codehumane/cache-handling/commit/d4d07e4684d4421bb6e23db6b93dea09c7d490f5)를 참고.

# 정리

1. Apache Shiro를 사용하면서 동시성 이슈를 만남.
2. 기존 지식으로는 해결하지 못함.
3. 문서와 간단한 테스트 코드를 통해 `ODE` 를 파악했고, `SimpleSession`의 내부 동작도 살펴봄.
4. 이것만으로는 부족. Apache Shiro와 애플리케이션이 세션 데이터를 어떻게 관리하는지 구조적 측면에서 살펴봄.
5. 어느 정도 문제를 파악한 후에는, 동시성 문제를 일으키는 조건을 제거하려 노력함.
6. 하지만 작업 비용이 꽤 크다고 판단하여, 동시성 문제가 되는 데이터 자체를 `SimpleSession`에서 제거하기로 결정.
7. 대신, `SimpleSession`으로부터 분리된 세션 데이터는 직접 구현하여 관리.
8. 쓰레드 풀 방식에서 `ThreadLocal` 사용이 일으킬 수 있는 문제를 만나 버그픽스를 하기도.
