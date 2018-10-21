---
layout: post
title: DDIA (Replication)
summary: Designing Data-Intensive Application 책에서 Replication 부분을 따로 정리.
date: 2018-10-13 00:00:01 +0900
---

재미없을 줄 알았는데, 막상 읽어 보니 참 재미있음. 이렇게 블로그에 따로 기록까지 하게 됨. 여러 내용 중에서도, 레플리케이션과 파티셔닝을 이 공간에 정리. 먼저, 레플리케이션.

# Replication

데이터의 복제본을 네트워크에 연결된 여러 머신에 유지하는 것. 이유는 아래와 같음.

1. Read Scalability: 읽기 부하를 분배.
2. Falut Tolerance/High Availability: 일부 장비가 고장나더라도 읽기 요청을 계속 처리할 수 있음.
3. Latency: 거리가 먼 사용자들에게 지역적으로 가까운 데이터 센터를 제공하여 응답 지연을 낮춤.

데이터 변경을 노드 간에 복제하는 방법에는 3가지가 존재.

1. single-leader
2. multi-leader
3. leaderless

그 외에도 아래 5가지 고려할 요소들이 존재함.

1. Synchronous versus Asynchronous
2. Setting Up New Followers
3. Handling Node Ouatages
4. Implementation of Replication Logs
5. Problems with Replication Lag

## Single-Leader Replication

가장 흔한 접근법. active/passive 또는 master-slave 레플리케이션이라고도 알려짐. 각 노드는 모두 레플리카라고 불리며, 쓰기 요청을 받는 레플리카를 리더 레플리카, 리더의 변경 사항을 복제하는 레플리카를 팔로워 레플리카라고 부름.

1. leader replica, master, active
2. follower replicas, slaves, passive replicas, read replicas, secondaries, hot standbys

작동 방식은 "[Leader-based (master-slave) replication](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0501.png)" 참고. 우리가 흔히 알고 있는 방식.

## Multi-Leader Replication

master-master 또는 active/active 레플리케이션이라고도 불림. 이 방식은 복잡성이 높기 때문에, 일반적으로는 비합리적인 선택. 하지만, 리더가 둘 이상이면, 특정 리더가 문제가 있더라도, 여전히 쓰기 요청을 처리할 수 있는 등의 몇 가지 이점이 있음.

1. Performance: 쓰기 성능이 좋아짐. (읽기는 X)
2. Tolerance of datacenter outages: 특정 데이터 센터의 장애가 국소화 됨. 다른 데이터 센터는 여전히 쓰기 요청을 처리할 수 있음.
3. Tolerance of network problems: 데이터 센터 간의 네트워크는 주로 퍼블릭이지만, 비동기로 레플리케이션이 이뤄지므로, 일시적 문제는 어느 정도 견뎌냄.

### Handling Writes Conflicts

리더가 많은 것이 복잡성을 높이는 이유는 바로 충돌.

| A 데이터센터 마스터                              | B 데이터센터 마스터                          |
| ------------------------------------------------ | -------------------------------------------- |
| `insert into user (id, name) values (1, 'foo');` |                                              |
|                                                  | A로부터 레코드 `1`이 레플리케이션 됨         |
|                                                  | `update user set name = 'bar' where id = 1;` |
| `update user set name = 'baz' where id = 1;`     |                                              |
| B로부터 레코드 `1`이 레플리케이션 됨.            |                                              |

레코드 `1`의 `name` 값은 `bar`인가, `baz`인가? 이런 충돌을 해결하는 방법으로 몇 가지를 제시함.

1. Synchronous Write Replication: 쓰기 요청 처리 시, 레플리케이션까지 끝나고 응답하기. 느리고, 신뢰성 낮음.
2. Conflict Avoidance: 레코드 별로 쓰기 가능한 리더를 할당. 하지만, 이 할당이 바뀌는 경우 충돌 회피는 깨짐. (하지만 책 내용과 별개로, 좀 더 확인이 필요해 보임)
3. Converging Toward a Consistent State: LWW 방식으로 가장 높은 ID(타임스탬프 등)를 가진 데이터를 남기거나, 가장 높은 숫자가 부여된 노드의 데이터를 남김. 혹은, 어떻게든 값을 병합(예컨대, 1과 3이라는 값을 1/3으로 병합). 충돌 데이터를 따로 저장하고, 데이터 읽기 시 사용자에게 충돌 해결을 위임하기도.
4. Custom Conflict Resolution Logic: 애플리케이션 코드를 통해 충돌 해결 로직을 작성하는 것. 쓰기 시점 혹은 읽기 시점에 처리 가능.

### Multi-Leader Replication Topologies

복수 리더 레플리케이션을 구성하는 방법. "[Three example topologies in which multi-leader replication can be set up](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0508.png)" 그림이 이해에 도움이 됨. 각 설명은 생략하고, 몇 가지만 기록.

- all-to-all이 가장 일반적.
- MySQL은 circular 만을 기본으로 지원.
- circular와 star는 무한 루프 방지를 위해 각 쓰기 별로 식별자를 태깅.
- circular와 star는 복제 경로 상 노드 장애에 취약.
- All-to-all에서는 추월<sup>overtake</sup> 문제가 있음. "[With multi-leader replication, writes may arrive in the wrong order at some replicas](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0509.png)" 그림 참고. 이를 해결하기 위해, 버전 벡터 같은 것이 사용되기도 함.

## Leaderless Replication

> Databases with appropriately configured quorums can tolerate the failure of indivisual nodes without the need for failover. They can also tolerate individual nodes going slow, because requests don't have to wait for all n nodes to respond―they can return when w or r nodes have responded. These characteristics make databases with leaderless replication appealing for use cases that require high availability and low latency, and that can tolerate occasional stale reads.

Dynamo 데이터베이스가 이 모델을 사용한다고 함(이것도 모르고 썼음). Riak, Cassandra, Voldemort도 사용하기 시작.

### Writing to the Database When a node Is Down

leaderless는 failover가 필요 없음. 대신, 아래처럼 읽기와 쓰기 요청을 처리. "[A quorum write, quorum read, and read repair after a node outage](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0510.png)" 그림 참고.

1. 클라이언트는 쓰기 요청을 모든 레플리카에게 병렬로 보냄.
2. 이 중 한 노드가 업데이트로 인해 중단 되었다고 가정.
3. 그러면, 클라이언트는 나머지 정상 노드로부터만 응답을 받음.
4. 그러다가 중단 되었던 노드가 다시 클러스터로 복귀.
5. 클라이언트는 읽기를 위해 모든 레플리카에게 병렬로 요청을 보냄.
6. 중단 되었던 노드로부터는 오래된<sup>stale</sup> 데이터를 받을 수 있음.
7. 클라이언트는 버전 번호를 통해 어떤 값이 최신인지를 스스로 판단.

장애 노드가 정상화 되면, 이 노드는 최신 쓰기를 어떻게 따라잡을까?

1. Read Repair
   - 클라이언트가 읽기 요청을 여러 노드에 동시에 날렸다면,
   - 최신 데이터와 오래된 데이터를 감지할 수 있으며,
   - 오래된 데이터를 반환한 노드에게 최신 데이터를 보내 쓰기 요청.
   - 자주 읽히는 값일 때 잘 동작.
2. Anti-Entropy Process
   - 백그라운드 프로세스가 있고,
   - 레플리카 간의 데이터 차이를 찾아다니다가,
   - 누락된 데이터가 있으면 이를 복제함.
   - 리더 기반 레플리케이션에서의 레플리케이션 로그와 다르게,
   - 쓰기를 순서대로 복제하지 않음.
   - 따라서, 데이터가 복제가 심각할 정도로 지연될 수도 있음.

### Quorums for Reading and Writing

쓰기와 읽기에는 쿼럼<sup>quorums</sup> 제약이 따름.

> 전체 노드의 갯수가 n, 쓰기가 가능한 노드의 갯수를 w, 읽기가 가능한 노드의 갯수를 r이라고 할 때, w + r > n을 만족시켜야 함.

참고로, w와 r은 얼마나 많은 노드로부터 응답을 기다려야 하는지를 결정하는 값. 일반적으로 n은 홀수로 두며, w = r = (n + 1) / 2 (반올림)으로 설정. "[If w + r > n, at least one of the r replicas you read from must have seen the most recent successful write](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0511.png)" 그림처럼, 적어도 하나의 노드로부터 최신 데이터를 읽어 들일 수 있기 때문. 좀 더 유연하게, w + r ≦ n으로 설정할 수도 있음. 오래된<sup>stale</sup> 값을 읽어 들일 가능성은 높아지지만, 응답 지연이 낮아지고, 가용성이 높아짐.

만약, 쿼럼을 만족시키지 못하면 에러를 반환하는 대신 아래와 같이 할 수도 있음.

1. sloppy quorum
   - n개 이외의 별도의 노드를 두고,
   - 이 별도의 노드까지 포함하여 w를 만족시킨다면,
   - 에러가 아닌 정상을 응답.
2. hinted handoff
   - 이렇게 별도의 노드가 받은 임시 쓰기를,
   - 다시 집<sup>home</sup>(원래의 n개에 포함되는) 노드로 돌려보냄.

Dynamo에서는 옵션, Riak에서는 기본, 그리고 Cassandra와 Voldmort에서는 비활성화가 기본.

### Detecting Concurrent Writes

쿼럼이 엄격한지 여부와 관계 없이 충돌은 발생할 수 있다. "[Concurrent writes in a Dynamo-style datastore: there is no well-defined ordering](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0512.png)" 그림 함께 참고. 이를 해결하는 방법은 몇 가지가 있음.

1. Last Write Wins
2. The "Happens-Before" Relationship and Concurrency: LWW와 다르게, 시간이 아닌, 의존성을 기준으로 판단.
3. Capturing the Happens-Before Relationship: "[Capturing causal dependencies between two clients concurrently editing a shopping cart](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0513.png)" 그림 참고. 이렇게 하면 버전 번호만을 통해 동시 작업 여부 감지와 병합이 가능.
4. Merging Concurrently Written Values: 3번과 유사한데, 삭제된 아이템에 버전 번호와 함께 마커를 남김. 이런 마커를 tombstone이라 부르고, 병합에 활용함.
5. Version Vectors: 3번과 동일한데, 여러 레플리카가 있는 경우를 고려한 것. 키 별로, 레플리카 별로 버전 번호를 사용.

참고로, 아래의 동시성<sup>concurrency</sup> 정의가 인상 깊었음.

> For defining concurrency, exact time doesn't matter: we simply call two operations concurrent if they are both unaware of each other, regardless of the physical time at which they occurred.

## Synchronous Versus Asynchronous Replication

많은 리더 기반 레플리케이션들은 완전한 비동기로 설정된다고 함. 이렇게 하면 쓰기의 신뢰성이 높아지고(실패할 가능성이 낮아짐), 지연 시간도 낮아짐. 다만, 내구성이 약해지는<sup>weakining durability</sup> 문제가 발생. 리더 노드가 고장났을 때, 최신 복제본을 가진 다른 노드가 있음을 보장하지 못하는 것. 한편, 책에서는 "[Leader-based replication with one synchronous and one asynchronous follower](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0502.png)" 그림처럼, semi-synchronous도 함께 설명.

## Setting Up New Followers

새로운 레플리카(책에서는 팔로워라고 했지만, 레플리카로 일반화 시켜도 될 것 같음)를 추가해야 할 때는 아래와 같이 한다고 함.

1. 지속적으로 리더 DB의 스냅샷을 생성.
2. 새로운 팔로워 노드로 스냅샷을 복사.
3. 팔로워는 리더에게 특정 스냅샷 이후로 생긴 모든 데이터 변경을 요청.
4. 모든 변경을 다 따라잡았다면 클러스터에 투입.

참고로, 데이터 변경점을 파악할 때, 레플리케이션 로그의 위치를 사용하는데, 이를 가리켜 log sequence number(PostgreSQL) 또는 bingo coordinates(MySQL)라고 부름.

## Handling Node Outages

노드가 고장나면 어떻게 해야 할까?

1. Follower Failure ― Catch-Up Recovery
   - 반영이 실패한 데이터 로그 지점부터 다시 반영을 시작.
   - 데이터 로그가 부족한 경우라면, 이를 리더로부터 받아오는 것부터 시작.
2. Leader Failure ― Failover
   - 리더의 고장 여부를 판단. 보통 타임아웃을 사용.
   - 선거<sup>election</sup>나 임명<sup>appoint</sup>을 통해 새로운 리더 선출(최신 데이터를 가진 레플리카가 좋은 후보).
   - 클라이언트가 바라보는 리더를 재설정.

하지만, 페일오버는 아래와 같은 문제를 가짐. 그래서, 어떤 운영팀은 수동 페일오버를 선호한다고 함.

1. 리더 노드에 장애가 났고, 비동기 레플리케이션이라면, 리더 데이터 일부가 유실될 수 있음.
2. 데이터 유실의 일반적인 해결책은 데이터 버림. 하지만 DB 데이터를 공유한 자원이 있다면 데이터 충돌.
3. 단일 리더를 기대하는 상황에서, 두 개의 노드가 모두 스스로를 리더라고 인식하기도.
4. 리더가 죽었는지를 판단하는 타임아웃의 적정 수준을 결정하기 어려움.

## Implementation of Replication Logs

레플리케이션은 어떻게 동작하는가?

1. Statement-Based Replication
   - 쓰기 명령문을 로깅하고 팔로워에게 보냄.
   - 하지만, `NOW()`, `RAND()`, 자동증가 컬럼 등이 포함된 비결정적 명령문은 위험.
2. Write-Ahead Log (WAL) Shipping
   - 말 그대로 WAL을 활용.
   - 하지만, 저수준의 데이터여서, 저장소와의 강한 결합. 심지어, 버전 차이가 문제 될 수도 있음.
3. Logical (Row-Based) Log Replication
   - WAL의 대안으로, 레플리케이션을 위한 별도의 로그 포맷을 사용.
   - 저장소 엔진의 물리적 데이터와 구분하기 위해 논리적 로그라고 함.
   - [MySQL에서의 binlog](https://dev.mysql.com/doc/internals/en/binary-log-overview.html)가 여기에 해당.
4. Trigger-Based Replication
   - 더 높은 유연성을 위해 애플리케이션 코드를 활용.
   - 코드로는 trigger 또는 stored procedure가 사용됨.

## Problems with Replication Lag

비동기 레플리케이션은 결과적 일관성을 보장하지만, 여기서의 결과적이라는 말은 모호함. 상황에 따라 수초에서 수분까지 걸릴 수 있음. 이에 따라 발생할 수 있는 문제와 해결책을 소개.

1. Read-your-writes / Read-after-write
   - 쓰기 요청이 아직 반영 되지 않은, 오래된<sup>stale</sup> 팔로워로부터 데이터를 읽어 들임.
   - 따라서, 자신이 수정할 수 있는 데이터라면 리더로부터 읽어 들이거나,
   - 갱신된지 1분 이내라면 리더에게 읽기 요청을 보냄.
2. Monotonic Reads
   - 같은 데이터에 대한 읽기 요청을 하더라도, 노드가 다르면 결과가 다를 수 있음.
   - 사용자 별로 지정된 노드로 요청을 라우팅하면, 비록 최신 데이터는 아닐지라도,
   - 한 번 읽어 들인 시점보다 과거의 데이터를 보는 문제는 막을 수 있음.
   - 강한 일관성<sup>strong consistency</sup>과 결과적 일관성<sup>eventual consistency</sup>의 중간.
3. Consistent Prefix Reads
   - A → B의 인과관계를 가지지만, 마치 B → A인 것처럼 읽히는 현상.
   - "[If some partitions are replicated slower than others, an observer may see the answer before they see the question](https://www.safaribooksonline.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0505.png)" 그림 함께 참고.
   - 이는 파티션 간에 전역 순서를 보장할 수 없기 때문에 발생하는 문제.
   - 관련 있는 데이터는 같은 파티션에 두는 등 쓰기가 발생한 순서대로 데이터를 읽게 해야 함.

## 마치며

레플리케이션은 매우 간단할 줄 알았으나, 생각보다 많은 양을 다루고 있어서, 내가 참 모르고 있었구나를 깨닫게 됨. 레플리케이션을 사용할 때는 그 목적과 한계를 잘 이해해야 한다고 생각함. 극단적으로는 레플리케이션을 단지 복구용이나 간단한 데이터 확인(운영상의) 용도로만 제한하는 것도 좋음. 대신, 파티셔닝을 사용. 물론, 레플리케이션의 한계가 애플리케이션에 주는 영향이 작은 경우도 있었음. 이럴 때는 부하를 분산하는 좋은 방법이라고 생각됨. 레플리케이션은 여기서 마치고, 파티셔닝에 대해 살펴볼 예정.