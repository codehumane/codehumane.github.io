---
layout: post
title: RedisHash TTL 설정과 키 명명 관례
summary: Spring Data Redis를 사용하며 겪은 RedisHash의 TTL 설정 문제와 그 해결
date: 2019-01-27 00:00:01 +0900
---

## 문제

[Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/2.1.4.RELEASE/reference/html/#redis.repositories)의 [@RedisHash](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/RedisHash.html)를 사용하는데, 여기서의 [timeToLive](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/RedisHash.html#timeToLive--) 설정이 제대로 먹지 않는 문제를 겪음. 예컨대, 아래의 데이터 구조가 있다고 해보자.

```java
@Value
@RedisHash(value = "BookRepresentation:v1", timeToLive = 15)
public class BookRepresentation {
    
    @Id
    private final String isbn;
    
    @Indexed
    private final String title;
    
    // ...
}
```

이 데이터가 캐시된 뒤, [redis-cli](https://redis.io/topics/rediscli)를 통해 데이터를 확인해 보면 아래와 같음.

```shell
127.0.0.1:6379> keys *
1) "BookRepresentation:v1"
2) "BookRepresentation:v1:9788992825764:phantom"
3) "BookRepresentation:v1:9788992825764"
4) "BookRepresentation:v1:title:hello"
```

참고로, 여기서 각 키의 의미는 아래와 같음.

| 키                                          | 설명                    |
| ------------------------------------------- | ----------------------- |
| BookRepresentation:v1:9788992825764         | 캐싱 대상 그 자체       |
| BookRepresentation:v1:9788992825764:phantom | 캐싱 데이터의 복제본    |
| BookRepresentation:v1                       | 캐시된 데이터의 키 집합 |
| BookRepresentation:v1:title:hello           | 인덱스 데이터           |

ttl을 15초로 설정해 두었으니, 15초가 지난 뒤 캐시 데이터를 살펴봄.

```shell
127.0.0.1:6379> keys *
1) "BookRepresentation:v1"

127.0.0.1:6379> smembers BookRepresentation:v1
1) "9788992825764"

127.0.0.1:6379> smembers BookRepresentation:v1:title:hello
1) "9788992825764"
```

기대와 다르게 `BookRepresentation:v1`, `BookRepresentation:v1:title:hello` 의 값이 지워지지 않음. 계속 쌓이기만 함. 여러가지 문제가 예상됨. (ㅠㅠ)

## 원인

왜 그런가를 찾아보고자 [`KeyExpirationEventMessageListener`](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/listener/KeyExpirationEventMessageListener.html)를 시작점으로 간단히 코드를 따라가 봄. 문제의 지점은 [`RedisKeyValueAdapter#onMessage`의 783L](https://github.com/spring-projects/spring-data-redis/blob/master/src/main/java/org/springframework/data/redis/core/RedisKeyValueAdapter.java#L783).

```java
RedisKeyExpiredEvent event = new RedisKeyExpiredEvent(channel, key, value);

ops.execute((RedisCallback<Void>) connection -> {
    connection.sRem(converter.getConversionService().convert(event.getKeyspace(), byte[].class), event.getId());
    new IndexWriter(connection, converter).removeKeyFromIndexes(event.getKeyspace(), event.getId());
    return null;
});

publishEvent(event);
```

[`RedisKeyExpireEvent#getId`](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/RedisKeyExpiredEvent.html#getId--) 값을 [`RedisSetCommands#sRem`](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/connection/RedisSetCommands.html#sRem-byte:A-byte:A...-)에 인자로 넘겨주고 있음. 메서드 역할은 [SREM 커맨드](https://redis.io/commands/srem) 참고. 그런데, 여기서의 아이디가 `BookRepresentation:v1`이 아니고, `BookRepresentation`임. 잘못된 키. 따라서, 제거되지 않음.

왜 이렇게 예상과 다른 키를 반환하는지는 `BinaryKeyspaceIdentifier#extractId`를 보면 이유를 알 수 있음.

```java
/**
* Parse a binary {@code key} into {@link BinaryKeyspaceIdentifier}.
*
* @param key the binary key representation.
* @return {@link BinaryKeyspaceIdentifier} for binary key.
*/
public static BinaryKeyspaceIdentifier of(byte[] key) {

    Assert.isTrue(isValid(key), String.format("Invalid key %s", new String(key)));

    boolean phantomKey = ByteUtils.startsWith(key, PHANTOM_SUFFIX, key.length - PHANTOM_SUFFIX.length);

    int keyspaceEndIndex = ByteUtils.indexOf(key, DELIMITTER);
    byte[] keyspace = extractKeyspace(key, keyspaceEndIndex);
    byte[] id = extractId(key, phantomKey, keyspaceEndIndex);

    return new BinaryKeyspaceIdentifier(keyspace, id, phantomKey);
}
```

`DELIMITER` 값이 `:`이고, 첫 번째 `:`를 기준으로 keyspace와 id를 나누고 있음. [An introduction to Redis data types and abstractions](https://redis.io/topics/data-types-intro) 문서의 [Redis Key](https://redis.io/topics/data-types-intro#redis-keys) 부분을 보니, 일종의 관례(명시적이지는 않다)인 것 같다.

> Try to stick with a schema. For instance "object-type:id" is a good idea, as in "user:1000". Dots or dashes are often used for multi-word fields, as in "comment​:1003:​reply.to" or "comment​:1003:​reply-to".

스프링은 이걸 기반으로 아이디를 쪼갠 것 같고.

## 해결

해결은 간단함. KeySpace를 기존의 `BookRepresentation:v1` 대신, `BookRepresentation-v1` 같은 형태로 바꿔주면 됨. 아래와 같이 말이다.

```java
@Value
@RedisHash(value = "BookRepresentation-v1", timeToLive = 15)
public class BookRepresentation {
    
    @Id
    private final String isbn;
    private final String title;
    // ...
}
```

## 생각

`@RedisHash`의 keySpace 설정부분에 약간의 설명을 달아줬으면 어땠을까 하는 아쉬움. 아니면, 형식에 대한 유효성 검증을 넣어주던가. 혹은 경고 로그라도. 아니면, `BinaryKeyspaceIdentifier`의 구현시, `@RedisHash`의 `keySpace`에 명시된 값을 참고했다면 어땠을까.

## 주의

TTL을 사용하려면 2가지를 주의해야 함.

1. `@EnableRedisRepositories`의 [`enableKeyspaceEvents`](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/repository/configuration/EnableRedisRepositories.html#enableKeyspaceEvents--) 활성화 필요함.
2. [Redis Keyspace Notifications](https://redis.io/topics/notifications) 값 설정이 되어 있지 않거나, 적어도 `Ex`를 포함한 상태로 설정되어 있어야 함.

`@EnableRedisRepositories`에는 `enableKeyspaceEvents` 속성이 존재. 기본 값은 OFF. TTL을 사용하려면 이 값을 ON_STARTUP 또는 ON_DEMAND 로 설정해야 함. 이렇게 하면, [`RedisKeyValueAdapter#initKeyExpirationListener`](https://github.com/spring-projects/spring-data-redis/blob/master/src/main/java/org/springframework/data/redis/core/RedisKeyValueAdapter.java#L706)이 실행되면서 [`MappingExpirationListener`](https://github.com/spring-projects/spring-data-redis/blob/master/src/main/java/org/springframework/data/redis/core/RedisKeyValueAdapter.java)를 등록하게 되고, 이 리스너는 다음의 2가지 일을 수행.

1. `notify-keyspace-events` 값을 적절히 설정.
2. 레디스가 보내준 expired 이벤트를 받아, 관련된 phantom, index, keyspace element 삭제.

첫 번째 일은 [`KeyspaceEventMessageListener#init`](https://github.com/spring-projects/spring-data-redis/blob/master/src/main/java/org/springframework/data/redis/listener/KeyspaceEventMessageListener.java#L81)를 보면 된다.

```java
/**
    * Initialize the message listener by writing requried redis config for {@literal notify-keyspace-events} and
    * registering the listener within the container.
    */
public void init() {

    if (StringUtils.hasText(keyspaceNotificationsConfigParameter)) {

        RedisConnection connection = listenerContainer.getConnectionFactory().getConnection();

        try {

            Properties config = connection.getConfig("notify-keyspace-events");

            if (!StringUtils.hasText(config.getProperty("notify-keyspace-events"))) {
                connection.setConfig("notify-keyspace-events", keyspaceNotificationsConfigParameter);
            }

        } finally {
            connection.close();
        }
    }

    doRegister(listenerContainer);
}
```

이 때의 `keyspaceNotificationsConfigParameter` 값은 `@EnableRedisRepositories`의 속성 값이며, 기본 값은 `Ex`. 이 값이 뭔지에 대해서는 [Redis Keyspace Notifications](https://redis.io/topics/notifications) 문서 참고.

두 번째 일은 `MappingExpirationListener#onMessage`를 살펴보면 됨.

```java
/*
    * (non-Javadoc)
    * @see org.springframework.data.redis.listener.KeyspaceEventMessageListener#onMessage(org.springframework.data.redis.connection.Message, byte[])
    */
@Override
public void onMessage(Message message, @Nullable byte[] pattern) {

    if (!isKeyExpirationMessage(message)) {
        return;
    }

    byte[] key = message.getBody();

    byte[] phantomKey = ByteUtils.concat(key, converter.getConversionService().convert(KeyspaceIdentifier.PHANTOM_SUFFIX, byte[].class));

    Map<byte[], byte[]> hash = ops.execute((RedisCallback<Map<byte[], byte[]>>) connection -> {

        Map<byte[], byte[]> hash1 = connection.hGetAll(phantomKey);

        if (!CollectionUtils.isEmpty(hash1)) {
            connection.del(phantomKey);
        }

        return hash1;
    });

    Object value = converter.read(Object.class, new RedisData(hash));

    String channel = !ObjectUtils.isEmpty(message.getChannel())
            ? converter.getConversionService().convert(message.getChannel(), String.class) : null;

    RedisKeyExpiredEvent event = new RedisKeyExpiredEvent(channel, key, value);

    ops.execute((RedisCallback<Void>) connection -> {

        connection.sRem(converter.getConversionService().convert(event.getKeyspace(), byte[].class), event.getId());
        new IndexWriter(connection, converter).removeKeyFromIndexes(event.getKeyspace(), event.getId());
        return null;
    });

    publishEvent(event);
}
```

phantom 데이터를 지우고, keyspace의 엘리먼트 중 관련 키를 지우고, 인덱스 데이터를 지우고 있음. 마지막으로, `RedisKeyExpiredEvent`를 만들어 다른 곳으로 전파까지. 참고로 phantom 데이터의 존재 이유는 아래와 같음.

> When the expiration is set to a positive value, the corresponding `EXPIRE` command is executed. In addition to persisting the original, a phantom copy is persisted in Redis and set to expire five minutes after the original one. This is done to enable the Repository support to publish `RedisKeyExpiredEvent`, holding the expired value in Spring’s `ApplicationEventPublisher`whenever a key expires, even though the original values have already been removed. Expiry events are received on all connected applications that use Spring Data Redis repositories.

