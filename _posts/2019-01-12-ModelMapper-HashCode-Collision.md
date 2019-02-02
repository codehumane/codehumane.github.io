---
layout: post
title: ModelMapper 해시코드 충돌과 오픈 소스 기여
summary: ModelMapper에서 발견한 결함과 생각, 그리고 오픈 소스 기여까지
date: 2019-01-12 00:00:01 +0900
---

[ModelMapper](https://github.com/modelmapper/modelmapper)를 사용하는데 간헐적으로 엉뚱한 필드를 매핑하려는 오류를 만남. 오류 원인을 추적하고자 ModelMapper 안에 있는 [`TypeMapStore`](https://github.com/modelmapper/modelmapper/blob/master/core/src/main/java/org/modelmapper/internal/TypeMapStore.java)에서 `typeMaps` 필드 내용을 로그로 남김. 로그 내용을 보면 필드 매핑이 잘못되어 있음을 알게 됨. 예컨대, `Source` 클래스의 인스턴스를 `Destination` 타입으로 변환한다고 해보자. 두 클래스는 각각 `name`,  `age`,  `sex`,  `phone` 속성을 가지고 있음. 이 때의 매핑 정보가 아래와 같은 식으로 되어 있는 것.

```
codehumane.Source -> codehumane.Destination
  PropertyMapping[Source.name -> Destination.name]
  PropertyMapping[Source.age -> Destination.age]
  PropertyMapping[Hello.hi -> Destination.sex] // Source.age 대신 엉뚱한 정보가 들어 있다.
  PropertyMapping[Source.phone -> Destination.phone]
```

그래서, source 클래스의 필드 정보를 어떻게 만드나 궁금해서 코드를 따라가 봄. 아래는 대략적인 코드의 호출 흐름.
1. `ImplicitMappingBuilder#new`
2. `TypeInfoImpl#getAccessors`
3. `PropertyInfoSetResolver.resolveAccessors`
4. `PropertyInfoResolver#propertyInfoFor`
5. `PropertyInfoRegistry.accessorFor`

이 중에서 문제가 된다고 생각하는 부분은 `PropertyInfoRegistry.accessorFor`. 코드를 보면 아래와 같음.

```java
// 중략 ...
private static final Map<Integer, Accessor> ACCESSOR_CACHE = new ConcurrentHashMap<Integer, Accessor>();

// 중략 ...
static synchronized Accessor accessorFor(Class<?> type,
                                         Method method,
                                         Configuration configuration,
                                         String name) {
    
  Integer hashCode = hashCodeFor(type, name, configuration);
  Accessor accessor = ACCESSOR_CACHE.get(hashCode);
  if (accessor == null) {
    accessor = new MethodAccessor(type, method, name);
    ACCESSOR_CACHE.put(hashCode, accessor);
  }

  return accessor;
}

// 중략 ...
private static Integer hashCodeFor(Class<?> initialType,
                                   String propertyName,
                                   Configuration configuration) {
    
  int result = 31 + initialType.hashCode();
  result = 31 * result + propertyName.hashCode();
  result = 31 * result + configuration.hashCode();
  return result;
}

// ...
```

`hashCodeFor`의 결과는 중복될 수 있음. 그래서 저 값을 키로 사용하면, 키가 중복될 경우 값이 덮어 씌워질 위험이 있음. 간단히 코드를 작성해 보면 이를 확인할 수 있음.

```java
@Test
public void hashCollision() {
    val key1 = new Key("aaa");
    val key2 = new Key("bbb");
    val key1HashCode = key1.hashCode();
    val key2HashCode = key2.hashCode();

    /*
     * hashCode 결과가 같은 Key 인스턴스를 키로 하여, 맵에 저장.
     * 해시 충돌 해결이 잘 되는 경우
     */
    val valid = new HashMap<Key, String>();
    valid.put(key1, "AAA");
    valid.put(key2, "BBB");
    assertEquals("AAA", valid.get(key1));
    assertEquals("BBB", valid.get(key2));

    /*
     * hashCode 결과를 키로 사용하는 경우.
     * 값은 다른데, hashCode 결과는 얼마든지 같을 수 있음.
     * 그런데도, 이걸 그대로 키로 사용하면 문제가 됨.
     */
    val invalid = new HashMap<Integer, String>();
    invalid.put(key1HashCode, "AAA");
    invalid.put(key2HashCode, "BBB");
    assertEquals("AAA", invalid.get(key1HashCode)); // 여기서 "BBB"가 반환됨
    assertEquals("BBB", invalid.get(key2HashCode));
}
```

`Integer`를 그대로 키로 쓰지 말고, 아래와 같이 하는 게 어땠을까.

```java
@Getter
@RequiredArgsConstructor
class PropertyKey {

    private final Class<?> initialType;
    private final String propertyName;
    private final Configuration configuration;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        PropertyKey that = (PropertyKey) o;
        return Objects.equals(initialType, that.initialType) &&
                Objects.equals(propertyName, that.propertyName) &&
                Objects.equals(configuration, that.configuration);
    }

    @Override
    public int hashCode() {
        int result = 31 + initialType.hashCode();
        result = 31 * result + propertyName.hashCode();
        result = 31 * result + configuration.hashCode();
        return result;
    }
}

class PropertyInfoRegistry {

    // 중략 ...
    private static final Map<PropertyKey, Accessor> ACCESSOR_CACHE = new ConcurrentHashMap<>();

    // 중략 ...
    static synchronized Accessor accessorFor(Class<?> type,
                                             Method method,
                                             Configuration configuration,
                                             String name) {
                                             
        final PropertyKey key = new PropertyKey(type, name, configuration);
        Accessor accessor = ACCESSOR_CACHE.get(key);
        if (accessor == null) {
            accessor = new MethodAccessor(type, method, name);
            ACCESSOR_CACHE.put(key, accessor);
        }

        return accessor;
    }

    // 중략 ...
}
```

아래는 해시 충돌에 관한 [Java API(Hashtable) 문서](https://docs.oracle.com/javase/10/docs/api/java/util/Hashtable.html)의 일부 설명.

> in the case of a "hash collision", a single bucket stores multiple entries, which must be searched sequentially

그리고 `HashMap#get`을 열어 보면 어떻게 동작하는지 어느 정도 알 수 있음.

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

## 업데이트

1. 01/12: [간단한 테스트 코드와 함께 PR도 제출](https://github.com/modelmapper/modelmapper/pull/438). 잘 돼서, 서비스 운영의 번거로움을 덜 수 있기를.
2. 01/20: merged!!

![github merged history](/images/2019-01-12-ModelMapper-HashCode-Collision/01-merged.png)