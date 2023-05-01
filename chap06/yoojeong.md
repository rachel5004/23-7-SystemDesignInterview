## 6장: 키-값 저장소 설계

- 키 값 데이터베이스라고도 불리는 비 관계형 데이터베이스
- 키 값은 고유 식별자여야 하며, 짧을수록 좋기 때문에 해시값을 사용하기도 한다

Q. Key 설계 어떻게 하시나요?

객체의 경우 hashcode 를 오버라이딩해서 쓸때도 있고 그냥 파라미터 조합으로 키를 쓰기도함
String으로 할 때는 메모리를 너무 먹어서 해시로 바꿈

---

```
- key-value의 크기는 10kb 이하
- 큰 데이터 저장 가능
- HA, 장애 상황에도 빨리 응답해야함
- autoscaling
- 데이터 일관성 수준은 조절 가능해야함
- latency 최소화
```

### 단일 서버 키-값 저장소

키-값 전부를 메모리에 해시 테이블로 저장으로 구현이 쉽고 빠른 속도 보장
가용 메모리 크기에 따라 데이터 압축(compression) 이나 자주 쓰이지않는 데이터는 디스크에 저장하는 방법을 쓰기도 함

### 분산 키 값 저장소

키 값 쌍을 여러 서버에 분산시키기 때문에 분산 해시 테이블이라고도 불림

#### CAP 정리

- Consistency (데이터 일관성) : 어느 노드에 접속하더라도 동일한 데이터를 제공
- Availability (데이터 가용성) : 일부 노드에 장애가 발생하더라도 데이터를 제공
- Partition Tolerance (파티션 감내) : 노드간 네트워크 단절이 있더라도 데이터를 제공

현실세계 분산시스템

- CP 시스템 : 데이터 가용성을 포기 (일관성을 강조한다. 가장 보편적인 시스템 - RDB)
- AP 시스템 : 데이터 일관성을 포기 (가용성을 강조한다. 일부 테이터가 유실되더라도 서비스 가능 - NoSQL)
- CA 시스템 : 네트워크 장애(P)는 피할 수 없는 문제로 여겨져 항상 Partition Tolerance를 지원하도록 설계되어야 함. (현실세계에서는 존재하지 않는다.)

### 시스템 컴포넌트

#### 데이터 파티션(partition)

데이터를 파티션 단위로 나눌때는 다음 요소를 고려해야한다.

- 데이터를 여러 서버에 고르게 분산할 수 있는가
- 노드를 추가 하거나 삭제할 때 데이터의 이동을 최소화할 수 있는가

안정 해시(consistent hash)는 이런 문제를 푸는 적합한 기술이다.

오토스케일링과 다양성을 유지할 수 있다고 나옴
근데 안정해시가 아니라도 오토스케일링은 되지 않나?
오토스케일링을 적용할 수 있다기보다는 오토스케일링 적용시에 재배치 문제를 쉽게 해결하는 것인듯

#### 데이터 다중화(replication)

높은 가용성을 위해 데이터를 N개 서버에 비동기적으로 다중화하여 저장한다.

키를 해시 링 위에 배치하고 시계 방향으로 링을 순회하여 만나는 첫 N개 서버에 데이터 사본을 보관한다.

```
N=3
s0 (key0)-> s1 -> s2 -> s3 --> ...
```

이 방법은 총 필요한 실제 물리서버가 부족할 수 있으므로 물리서버를 중복해서 선택하지 않도록 해야한다.

#### 데이터 일관성(consistency)

다중화된 노드의 데이터를 동기화를 위해 정족수 합의(Quorum Consensus) 프로토콜을 사용한다.

이 프로토콜은 3가지 값이 필요하다.

- N : 데이터 노드 수
- W : 쓰기 연산에 대한 정족수, W개 이상의 서버로부터 쓰기가 성공했다는 응답을 받아야함
- R : 읽기 연산에 대한 정족수, R개 이상의 서버로부터 읽기가 성공했다는 응답을 받아야함
  R=1, W=N : 읽기 연산에 최적화
  W=1, R=N : 쓰기 연산에 최적화
  W+R > N : 강한 일관성이 보장됨

(inconsistency resolution)

#### 장애 처리

`장애감지`: 두 대 이상의 서버가 특정 서버의 장애를 보고하면 장애로 간주

| 종류 | Centralized                                                                                                             | Spanning Tree                                                                                                                                                                                                                                                | Gossip                                                                                                                                                                                                |
| :--: | :---------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 구현 | 하나의 중앙 컨트롤러(Central Controller)가 모든 Multicast 그룹의 멤버들을 관리하고 모든 메시지를 중앙에서 분배          | Spanning Tree 프로토콜을 사용하여 Multicast 그룹에 속한 모든 노드들을 트리 구조로 연결                                                                                                                                                                       | 각 노드가 주기적으로 메시지를 무작위로 선택한 다른 노드들에게 전파                                                                                                                                    |
| 장점 | 구현이 쉽다                                                                                                             | 각 노드들은 O(1)만큼의 과부하를 받고 각 노드들은 O(log(N))의 Latency를 보장할 수 있다                                                                                                                                                                        | 약 Log(N)의 시간안에 (낮은 Latency) 거의 대부분의 노드들에게 데이터가 전송되며 (1 - 1 / (N ^ (cb - 2)) 노드들은 약 Log(N)의 Load만을 받게 된다. 이 모든 특성을 UDP로만 이룰 수 있기때문에 더욱 효율적 |
| 단점 | Central노드가 Fail할 시 모든 노드들이 데이터를 받지 못하며 다른 모든 노드와 통신을 하기 때문에 O(N)의 Load를 받게 된다. | 한 노드에 장애가 생기면 하위 노드들이 데이터를 받지 못한다.<br> 이를 극복하기 위해 RTMP(Reliable Multicast Transport Protocol)에서는 데이터를 성공적으로 받았을때 Ack보내는 방법을 사용하지만, O(N)의 Ack Load를 Root노드에 부과하기 때문에 강점이 사라진다. | 각 노드가 메시지를 전달하는 주기를 적절히 설정해야 한다.                                                                                                                                              |

`일시적 장애 처리`

엄격한 정족수 접근법 : 읽기와 쓰기 금지
느슨한 정족수 접근법 : 장애 서버를 제외하고 다시 W,R개의 서버를 고름

장애 서버 복구까지 다른 서버가 처리하고, 복구 완료 시 남겨둔 힌트로 일괄 반영하는 것을 "hinted handoff" 라고 함

`영구적 장애 처리`

anti-entropy protocol :

`데이터센터 장애`

같은 센터 내의 데이터는 장애를 동시에 겪을 가능성이 있으므로 데이터센터별로 다중화하는 것이 좋다

#### 시스템 아키텍쳐

#### 읽기 경로(read path)

#### 쓰기 경로(write path)

---

|            목표 / 문제            | 기술                                       |
| :-------------------------------: | :----------------------------------------- |
|        대규모 데이터 저장         | 안정 해시를 사용해 서버들에 부하 분산      |
| 읽기 연산에 대한 높은 가용성 보장 | 데이터를 여러 데이터센터에 다중화          |
| 쓰기 연산에 대한 높은 가용성 보장 | 버저닝 및 벡터 시계를 사용한 충돌 해소     |
|           데이터 파티션           | 안정 해시                                  |
|        점진적 규모 확장성         | 안정 해시                                  |
|              다양성               | 안정 해시                                  |
|     조절 가능한 데이터 일관성     | 정족수 합의                                |
|         일시적 장애 처리          | 느슨한 정족수 프로토콜과 단서 후 임시 위탁 |
|         영구적 장애 처리          | 머클 트리                                  |
|       데이터 센터 장애 대응       | 여러 데이터 센터에 걸친 데이터 다중화      |