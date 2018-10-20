---
layout: post
title: DDIA (Partitioning)
summary: Designing Data-Intensive Application 책에서 Partitioning 부분을 따로 정리.
date: 2018-10-13 00:00:01 +0900
---

# Partitioning

이번에는 파티셔닝에 대해 정리. 데이터 셋이 매우 크거나, 쿼리 처리량이 매우 높아야 하는 경우에는 레플리케이션 만으로 한계. 그래서, 데이터를 여러 파티션으로 분할하게 됨. 샤딩(용어의 정의는 찾아 보면 조금씩 다른 것 같다)이라고도 부름. 보통, 하나의 노드 안에 여러 파티션이 위치하게 됨.

## Partitioning and Replication

1. 레플리케이션과 파티셔닝은 서로 (거의) 독립적.
2. 따라서, 이 둘의 다양한 조합이 가능함.
3. "[Combining replication and partitioning: each node acts as leader for some partitions and follower for other partitions](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0601.png)" 그림 참고.

## Partitioning of Key-Value Data

파티셔닝의 목적 중 하나가 부하 분산. 그런데 데이터가 적절히 분배되지 못한다면, 특정 파티션으로 부하가 집중될 수 있음. 이런 불균형을 가리켜 *skewed*, 부하가 집중된 파티션을 가리켜 *hot spot*이라고 부름. 결국, 복잡성만 늘어나는 것. 따라서, 데이터를 저장할 노드를 결정하는 기준이 중요. 어떤 기준을 선택해야 할까?

### Random

1. 데이터를 파티션에 무작위로 연결시킬 수 있음.
2. 이렇게 하면 데이터가 고르게 분배될 것.
3. 그러나, 이렇게 되면 데이터를 찾기 위해 매번 모든 노드에 질의를 던져야 함.
4. 지연시간 증폭 등의 문제가 있어 비효율적.

### Key Range

1. 백과사전이 알파벳 기준으로 여러 권으로 나뉘듯,
2. 타임스탬프 등을 기준으로 연속적인 구간을 나누어 파티션에 데이터를 할당.
3. 예를 들어, 2016년 데이터는 1번 노드, 2017년은 2번, 2018년은 3번, ...
4. 이렇게 하면, 데이터가 어느 노드에 속하는지 알고, 노드를 지정해서 질의할 수 있음.
5. 다만, 시간이 지나면서 점점 불균형이 생길 수도.
6. 예를 들어, 2018년 7월에 사용자가 급증 → 특정 노드가 *hot spot*이 됨.

### Hash of Key

1. 이런 skew와 hot spot을 극복하고자 해시 함수를 많이 사용.
2. 이 접근법에 대한 설명은 "[Partitioning by hash of key](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0603.png)" 그림을 참고.
3. 만약, 32 비트의 해시 함수를 사용하고, 파티션이 8개라면, 해시 값을 2^29 단위로 나눌 수 있을 것.
4. 0 ≤ hash(key) < b0, b0 ≤ hash(key) < b1, ...
5. 하지만, 데이터들이 순차적으로 분산되어 있지 않아 범위 검색이 어려움.

참고로, 해시 함수를 선택할 때의 고려사항은 아래와 같음.

1. 먼저, 암호화 수준이 높을 필요는 없음.
2. 그리고 `Object#hashCode` 등 프로그래밍 언어에서 제공하는 해시 함수는 주의.
3. 같은 값에 대한 해싱 결과가 달라질 수 있기 때문.
4. [Java's hashCode is not safe for distributed system](https://martin.kleppmann.com/2012/06/18/java-hashcode-unsafe-for-distributed-systems.html) 함께 참고.
5. MongoDB는 MD5, Cassandra는 Murmur3, Vodemort는 Fowler-Noll-Vo를 해싱 함수로 사용.

또한, 범위 검색 문제에 대한 데이터베이스 별 대응은 아래와 같음.

1. MongoDB, ElasticSearch ― 범위 검색을 위해 모든 파티션에 요청을 보냄.
2. Riak, Couchbase, Voldemort ― 기본 키에 대한 범위 검색을 아예 지원 안 함.
3. Cassandra ― *coupound primary key* 제공. 첫 번째 키는 파티션 찾기에, 나머지 키는 파티션 내 범위 검색에 사용.

### Skewed Workloads and Relieving Hot Spots

1. 해시 방식을 사용한다고 하더라도, hot spot을 완전히 피할 수는 없는 일.
2. 이런 불균형 해소를 데이터베이스가 해주지는 않음.
3. 결국, 애플리케이션에서 직접 문제를 다뤄야 함.
4. 책에서는 hot spot을 일으키는 주요 키를 따로 관리하고,
5. 이 키에 한해서만 별도의 라우팅을 하라고 이야기 함.
6. 모든 키에 대해 이런 작업을 하는 것은 부담이므로.

## Partitioning and Secondary Indexes

아래는 보조 인덱스의 성격을 잘 나타내주는 문장.

> A secondary index usually doesn’t identify a record uniquely but rather is a way of searching for occurrences of a particular value

레코드의 유일한 식별을 보장하지 못함. 따라서, 레코드가 정확히 어느 파티션에 있는지 알기 어려움. 이런 문제 때문에 많은 키-값 저장소들은 보조 인덱스를 지원하지 않음(HBase, Voldemort, ...). 하지만, 보조 인덱스는 매우 유용하며, Riak 등의 일부 저장소는 보조 인덱스를 지원하기 시작. Solr와 Elasticsearch 등의 검색 서버에서는 보조 인덱스가 심지어 존재의 이유<sup>raison d’être</sup>.

파티셔닝에서 보조 인덱스를 활용하기 위한 접근법은 크게 2가지.

### Partitioning Secondary Indexes by Document

1. "[Partitioning secondary indexes by document](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0604.png)" 그림처럼,
2. 파티션 별로 보조 인덱스를 독립적으로 관리.
3. 이 때문에 로컬 인덱스라고 불리기도 함.
4. 독립적으로 구성되어 있기 때문에, scatter/gatter 방식으로 질의.
5. 모든 파티션에게 질의를 던지고 결과를 병합하는 것.
6. 이는 지연시간 증폭<sup>latency amplification</sup> 등의 문제를 수반.
7. 따라서, 단일 파티션에서 보조 인덱스 질의가 수행되도록 구성하는 것이 좋음.

### Partitioning Secondary Indexes by Term

1. 문서 대신 용어<sup>term</sup> 별로 인덱스를 파티셔닝하는 것. 글로벌 인덱스.
2. "[Partitioning secondary indexes by term](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0605.png)" 그림을 보면 이해가 쉬움.
3. 참고로, 용어는 문서에 존재하는 단어를 가리킴. 그러니까, 인덱싱의 후보들.
4. 파티션의 기준은 2가지.
   - 용어를 정렬한 뒤 구간 별로 나누어 파티셔닝 하거나(범위 검색이 쉬움).
   - 해싱 값을 기준으로 파티셔닝(고르게 분산됨).
5. 글로벌 인덱스에서는 모든 파티션에 매번 질의를 하지 않으므로 효율적.
6. 하지만, 쓰기가 복잡하고 느림(그림을 보면 이유를 알 수 있음).
7. 또한, 인덱싱 갱신이 비동기. 따라서, 지연 가능성 존재.

## Rebalancing Partitions

책에서는 리밸런싱을 다음과 같이 정의함.

> All ot these changes call for data and requests to be moved from one node to another. THe process of moving load from one node in the cluster to another is called *rebalancing*.

### Hash Mod N (How Not to Do It)

해시 파티셔닝 시, 해시 값을 노드 수로 나누어 나머지를 구하고, 이 나머지 값을 노드 번호에 대응시켜 데이터를 분산할 수도 있었음. 그런데, 왜 그렇게 하지 않을까?

1. 노드가 늘어나면(혹은 줄어들면), 대부분의 키를 다른 노드로 이동시켜야 하기 때문.
2. 예를 들어, hash(key) = 123456이라면, 10개의 노드일 때 mod 값은 6.
3. 그런데, 노드가 11개로 늘어나면, 이 키는 이제 3번 노드로 옮겨가야 함.
4. 12개로 늘어나면, 0번 노드로 이동해야 함.
5. 리밸런싱은 최소화하는 게 좋다.

### Fixed Number of Partitions

1. 노드 별로 여러 개의 파티션을 미리 할당.
2. 새로운 노드가 추가 되면, 각 노드에서 일부 파티션만을 새로운 노드로 재할당.
3. "[Adding a new node to a databse cluster with multiple partitions per node](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0606.png)" 그림 참고.
4. 노드 별로 이동해야 하는 데이터를 최소화 하는 것.
5. 이렇게 하면, 파티션의 갯수도 변경되지 않고(그래서 제목이 Fixed Number of Partitions),
6. 파티션에 대한 키 할당도 변경되지 않음.
7. 오직, 노드에 할당되는 파티션만 바뀜.
8. Riak, ElasticSearch, Couchbase, Voldemort가 이 방식을 사용.

### Dynamic Partitioning

1. Fixed Number of Partitions 방식에서는 특정 파티션에만 데이터가 몰릴 수 있음.
2. 이 경계를 매번 조정하는 것은 성가신 일.
3. 이를 없애고자 HBase나 RethinkDB는 파티션을 동적으로 관리.
4. 파티션의 크기가 일정 수준 이상으로 넘어가면 2개로 나눔.
5. 반대로, 파티션 크기가 작아지면 다른 파티션과 병합.
6. 이 방식은 파티션의 수가 전체 데이터 볼륨에 적응적(장점이다).
7. MongoDB도 2.4 이후로는 동적 분할을 지원.

### Partitioning Proportionally to Nodes

1. Dynamic Partitioning은 파티션의 수가 데이터 셋의 크기에 비례하고,
2. Fixed Number of Partitions에서는 파티션의 크기가 데이터 셋 크기에 비례했음.
3. 세 번째 옵션은 파티션 수를 노드의 수에 비례하게 하는 것.
4. 즉, 노드 별로 파티션의 수를 고정함.
5. 새 노드가 클러스터에 합류하면, 일정 수의 파티션을 무작위로 골라 반으로 나눈 뒤, 새 노드로 이동.
6. 무작위로 선택해서 불균형이 일어날 것 같지만, 여러 번 파티셔닝 하기 때문에 결과적으로 균형.
7. consistent hashing의 원래(?) 정의에 거의 근접하다고 함.
8. Cassandra, Ketama에서 사용하는 방식.

### Operations: Automatic or Manual Rebalancing

리밸런싱은 수동으로 하는 게 좋을까 자동으로 하는 게 좋을까?

1. 완전 자동화 된 방식은 물론 편함.
2. 하지만, 리밸런싱은 비용이 큰 작업. 요청을 라우팅해야 하고, 많은 양의 데이터를 옮겨야 함.
3. 이는 네트워크 부담을 가중시키고, 성능을 저하시킬 수 있음.
4. 게다가, 자동화된 장애 감지와 함께 사용되면, 장애 전파로 이어질 수도.
   - 예를 들어, 한 노드에 부하가 걸렸고,
   - 다른 노드들이 이 노드를 죽은 것으로 간주하게 되고,
   - 자동으로 리밸런싱이 시작되면,
   - 부하 걸린 노드에 더 큰 부하를 주게 됨.
   - 이는 네트워크에게도 추가적인 부담.
   - 즉, 장애 전파<sup>cascading failure</sup>.
5. 이런 예측하기 어려운 문제 때문에, 사람의 개입이 어느 정도 필요.

## Request Routing

각 파티션들은 노드에 분산되어 있음. 클라이언트는 요청을 보낼 때 누구에게 보내야 하는지를 어떻게 알 수 있을까? 일종의 service discovery.

1. 클라이언트는 임의의 노드에게 요청을 보냄. 데이터가 있으면 노드는 바로 응답하고, 그렇지 않으면 적절한 노드로 데이터를 포워딩. 포워딩한 노드에게 응답을 받으면, 이를 다시 클라이언트로 전달.
2. 라우팅 계층(partition-aware load balancer)이 따로 존재.
3. 클라이언트가 어느 노드에 파티션이 할당되어 있는지를 미리 파악. 혹은 규칙을 가지거나.

## 마무리

간단히 Replication과 Partitioning에 대해 살펴봄. 책을 통해, 잘 몰랐던 부분까지 전반적으로 알게 되어 너무 좋았음. 하지만, 책은 책일 뿐이라는 것을 또한 느끼게 됨. 책에서 문제라고 했지만, 실제 업무에서는 문제가 되지 않기도 하고, 현업에서 사용했던 방식이 책에서는 아예 다뤄지지 않기도 함. 매번 느끼는 거지만, 결국 애플리케이션의 특성을 잘 파악해야, 이런 지식도 의미있다고 느껴짐. 그럼에도 불구하고 재미있고 의미 있던 책.