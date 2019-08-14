---
layout: post
title: DDIA (Consistency and Consensus)
summary: Designing Data-Intensive Application 책에서 Conssitency and Consensus 부분 요약
date: 2019-08-14 00:00:01 +0900
---

Designing Data-Intensive Applications이라는 책의 9장 Consistency and Consensus을 읽었고, 이를 리마인드 하는 차원에서 맨 뒷 부분에 나오는 Summary를 요약하여 여기에 기록.

일단, linearizability에 대해 다룸.

- 유명한 일관성 모델.
- 마치 데이터가 단일 본<sup>single copy</sup>만 존재하는 것처럼 보이게 함.
- 또한 모든 연산을 원자적으로 처리.
- 하지만 느림.
- 네트워크 지연이 큰 환경일 수록 더더욱.

다음으로 causality에 대해 다룸.

- 이벤트에 순서를 부과하는 것.
- linearizability에 비해서는 약한 일관성 모델을 제공.
    - linearizability: totally ordered. single timeline.
    - causality: partial order(인과적 관계가 있는 경우에만 ordered). branching and merging timeline.
- 그만큼 네트워크 문제에 좀 더 덜 민감하고 오버헤드도 적음.
- Lamport timestamp 방식도 여기서 소개됨.

하지만 causal ordering 만으로는 안 되는 것들이 존재.

- username이 고유해야 하는 제약이 있고, 동시에 사용자 등록 요청이 발생한다면?
- 한 노드에서 등록 요청을 받았을 때, 다른 노드에서 같은 username에 대해 동시에 실행중인 작업이 없는지 확인해야 함.
- 네트워크 지연이 있거나 등등의 이유로 문제가 많이 발생할 수 있음.
- 참고로, 동시<sup>concurrent</sup>라는 용어의 정의도 명확히 하고 있으므로 책에서 확인하면 좋을 듯.
- 이 문제는 컨센서스로 이어짐.

컨세서스는 어떤 결정에 대해 모든 노드에 동의를 구하는 방식으로 뭔가를 결정하는 것. 그리고 이 결정은 바뀌지 않음<sup>irrevocable</sup>. 컨센서스를 통해 여러 문제를 해결할 수 있음. 아래와 같은 것들.

- Linearizable compare-and-set registers
- Atomic transaction commit (distributed)
- Total order broadcast
- Locks and leases. 
- Membership/coordination service
- Uniqueness constraint

이는 싱글 노드에서는 매우 단순한 문제들. 혹은 의사결정을 특정 노드에서만 할 때도 마찬가지. 싱글 리더 데이터베이스가 대표적인 예. 모든 결정은 리더가 하므로, linearizable operations, uniqueness constraints, a totally ordered replication log 등이 가능. 하지만 싱글 리더에 장애가 생기거나, 네트워크 간섭으로 인해 리더가 이용 불가하면, 시스템은 더 이상 일을 진행할 수 없음. 이런 상황을 다룰 수 있는 3가지 방법이 있음.

1. 리더의 복구를 기다림. 그리고 시스템이 일정 시간 블럭 될 수 있음을 받아들이기. 많은 XA/JTA 트랜잭션 코디네이터가 이 방식. 이는 컨센서스는 아님. *termination* 속성을 만족시키지 못하기 때문.
2. 수동 페일오버. 사람이 새로운 리더 노드를 결정하고 재설정하게 하는 것. 많은 관계형 데이터베이스들이 이 방식을 취함. "act of God에 의한 컨센서스. 자동 페일오버에 비해서는 느릴 수 밖에 없음.
3. 알고리즘을 이용하여 새로운 리더를 자동으로 선출. 컨센서스 알고리즘이 필요. 불안정한 네트워크 상황을 잘 대응할 수 있는 알고리즘을 선택하는 것이 중요.

ZooKeeper는 "아웃소싱된" 컨센서스, 장애 감지, 멤버십 서비스 등을 제공. 사용하기가 쉽지는 않지만 처음부터 모든 것을 구현해서 쓰는 것 보다는 더 나은 선택. 물론, 모든 시스템들에 컨센서스가 필요한 것은 아님. 서로 다른 리더간에 컨센서스를 이루지 못해 충돌이 발생할 수 있고 이를 허용하기도.

