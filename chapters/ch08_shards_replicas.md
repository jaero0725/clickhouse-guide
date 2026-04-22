# 8장. Shards와 Replicas — 분산 아키텍처

단일 서버의 한계에 도달하면 ClickHouse 클러스터를 구성한다. 이 장에서는 Shard(데이터 수평 분할), Replica(데이터 복제), Distributed 테이블(라우팅 레이어), 그리고 분산 쿼리 실행의 구조와 한계를 다룬다.

---

## 8.1 Shard: 데이터를 나누는 단위

### 샤딩이 필요한 두 가지 경우

1. **데이터가 단일 서버의 스토리지를 초과**할 때
2. **단일 서버의 처리 성능이 부족**할 때

샤딩은 테이블의 데이터를 여러 ClickHouse 서버로 **수평 분할**한다. 각 shard는 데이터의 부분 집합을 담는 일반적인 ClickHouse 테이블이며, 독립적으로 쿼리할 수 있다.

```
[전체 데이터]
   ├── Shard 1 (Server A): 행 1 ~ 1000만
   ├── Shard 2 (Server B): 행 1000만+1 ~ 2000만
   └── Shard 3 (Server C): 행 2000만+1 ~ 3000만
```

### Distributed 테이블: 데이터를 저장하지 않는 라우팅 레이어

개별 shard에 직접 쿼리하면 부분 데이터만 보인다. 전체 데이터셋에 대한 통합 뷰를 제공하는 것이 **Distributed 테이블**이다.

```sql
CREATE TABLE uk.uk_price_paid_simple_dist ON CLUSTER test_cluster
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = Distributed('test_cluster', 'uk', 'uk_price_paid_simple', rand())
```

핵심 특성:
- **데이터를 자체 저장하지 않는다** — 모든 로컬 테이블에 대한 "view"일 뿐
- **SELECT**: 모든 shard에 쿼리를 전달하고, 결과를 수집·병합
- **INSERT**: sharding key를 평가하여 적절한 shard로 행을 라우팅

### Sharding Key와 INSERT 라우팅

```
INSERT → Distributed 테이블
  ↓
각 행에 대해:
  ① sharding key 평가 (예: rand())
  ② 결과 % shard 수 = 대상 서버 ID
  ③ 해당 서버의 로컬 테이블에 INSERT
```

`rand()`는 무작위 분산을 보장한다. 하지만 특정 쿼리 패턴에 따라 `cityHash64(user_id)` 같은 결정적 키를 사용하면, 같은 사용자의 데이터가 같은 shard에 모여 JOIN이나 GROUP BY 효율이 높아질 수 있다.

---

## 8.2 분산 쿼리 실행

### SELECT 처리 흐름

```
① 클라이언트 → Distributed 테이블에 SELECT 전송

② Distributed 테이블이 모든 shard에 쿼리 전달
   → 각 서버가 로컬에서 독립적으로 쿼리 실행 (병렬)

③ Initiator 서버가 모든 로컬 결과를 수집

④ 결과를 병합하여 최종 글로벌 결과 생성

⑤ 클라이언트에 반환
```

공식 architecture.md의 설명:

> "The Distributed table requests remote servers to process a query just up to a stage where intermediate results from different servers can be merged. Then it receives the intermediate results and merges them."

핵심 설계 원칙: **가능한 많은 작업을 원격 서버에서 처리**하고, 네트워크로 전송하는 중간 데이터를 최소화한다.

### Global Query Plan이 없다

이것은 ClickHouse 분산 쿼리의 가장 중요한 구조적 제약이다:

> "There is no global query plan for distributed query execution. Each node has its local query plan for its part of the job. We only have simple one-pass distributed query execution: we send queries for remote nodes and then merge the results."

**1-pass 분산 실행**만 지원한다: 쿼리를 원격 노드에 보내고 결과를 머지. 그 이상의 조율은 없다.

### 분산 실행이 어려운 쿼리

다음 유형의 쿼리에서 이 제약이 문제가 된다:

**고카디널리티 GROUP BY**: 각 shard의 로컬 GROUP BY 결과가 방대하면, Initiator가 이를 머지할 때 메모리와 네트워크 부하가 급증한다.

**대용량 데이터 JOIN**: 두 개의 Distributed 테이블을 JOIN하면, 한쪽 테이블의 데이터를 다른쪽의 모든 shard에 브로드캐스트해야 할 수 있다.

**서브쿼리의 Distributed 테이블**: `IN` 또는 `JOIN` 절에 Distributed 테이블이 포함되면 실행 전략이 복잡해진다.

공식 문서는 이에 대해 솔직하다:

> "In such cases, we need to 'reshuffle' data between servers, which requires additional coordination. ClickHouse does not support that kind of query execution, and we need to work on it."

---

## 8.3 Replica: 데이터 복제

### 복제의 목적

1. **데이터 무결성 (Data Integrity)**: 하드웨어 장애 시 데이터 손실 방지
2. **페일오버 (Failover)**: 서버 장애 시 다른 레플리카에서 서비스 지속
3. **쿼리 처리량 (Throughput)**: 여러 레플리카에 쿼리를 분산하여 병렬 처리

### 복제의 구조

```
Cluster
  ├── Shard 1
  │     ├── Replica A (Server 1)
  │     ├── Replica B (Server 2)
  │     └── Replica C (Server 3)
  └── Shard 2
        ├── Replica A (Server 4)
        ├── Replica B (Server 5)
        └── Replica C (Server 6)
```

각 shard 내에서 모든 레플리카는 **동일한 데이터**를 보유한다. SELECT 쿼리 시 Distributed 테이블은 각 shard에서 **하나의 레플리카만 선택**하여 쿼리를 전달한다.

### ReplicatedMergeTree

복제는 `ReplicatedMergeTree` 스토리지 엔진으로 구현된다:

```sql
CREATE TABLE uk.uk_price_paid_simple ON CLUSTER test_cluster
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/uk_price_paid', '{replica}')
ORDER BY (town, street);
```

**같은 ZooKeeper 경로를 가진 모든 테이블이 서로의 레플리카**가 된다. 레플리카는 테이블 생성/삭제만으로 동적으로 추가·제거할 수 있다.

### 비동기 멀티마스터 복제

```
             ZooKeeper / ClickHouse Keeper
                    │
    ┌───────────────┼───────────────┐
    │               │               │
  Replica A      Replica B      Replica C
  (INSERT 가능)  (INSERT 가능)  (INSERT 가능)
```

핵심 특성:

**비동기(Asynchronous)**: 데이터가 어느 레플리카에든 INSERT된 후, 다른 레플리카에 비동기적으로 복제된다. INSERT 시점에 모든 레플리카의 확인을 기다리지 않는다.

**멀티마스터(Multi-master)**: ZooKeeper 세션이 있는 **어떤 레플리카에든 INSERT 가능**하다. 단일 마스터 병목이 없다.

**Conflict-free**: ClickHouse는 UPDATE를 지원하지 않으므로 (INSERT only + 머지 기반 변환), 복제 시 충돌이 발생하지 않는다.

### 복제 로그와 물리적 복제

ZooKeeper에 저장되는 **복제 로그(replication log)**가 복제를 조율한다. 로그에 기록되는 액션:

- `get part`: 파트 다운로드
- `merge parts`: 파트 머지
- `drop partition`: 파티션 삭제

각 레플리카는 복제 로그를 자신의 큐에 복사하고 순서대로 실행한다.

**물리적 복제(Physical Replication)**: 쿼리가 아닌 **압축된 파트 자체**가 노드 간 전송된다. 이는 네트워크 효율을 위한 설계다. 머지는 대부분의 경우 **각 레플리카에서 독립적으로 수행**되어 네트워크 증폭(network amplification)을 회피한다. 대용량 머지된 파트는 복제 지연(replication lag)이 심각할 때만 네트워크로 전송된다.

### 머지 조율

모든 파트가 모든 레플리카에서 **byte-identical**하게 머지되도록 조율한다:

1. **리더(leader)** 중 하나가 새 머지를 시작하고 복제 로그에 "merge parts" 액션을 기록
2. 다른 레플리카가 로그를 읽고 동일한 머지를 독립 수행
3. 결과가 byte-identical이므로 네트워크 전송 불필요

여러 레플리카(또는 전부)가 동시에 리더가 될 수 있다. `replicated_can_become_leader` 설정으로 특정 레플리카의 리더 역할을 방지할 수 있다.

### 일관성 복구

각 레플리카는 ZooKeeper에 자신의 상태(파트 목록 + 체크섬)를 저장한다.

```
로컬 파일시스템 상태 ≠ ZooKeeper 참조 상태
  → 누락/손상 파트를 다른 레플리카에서 다운로드하여 복구
  → 예상치 못한/손상된 데이터는 별도 디렉터리로 이동 (삭제하지 않음)
```

---

## 8.4 클러스터의 구조적 한계

공식 architecture.md의 경고:

> "The ClickHouse cluster consists of independent shards, and each shard consists of replicas. The cluster is **not elastic**, so after adding a new shard, data is not rebalanced between shards automatically."

### 핵심 한계 정리

**자동 리밸런싱 없음**: 새 shard를 추가해도 기존 데이터가 자동으로 재분배되지 않는다. 수동으로 데이터를 이동해야 한다.

**노드 간 데이터 재분배(reshuffle) 미지원**: 복잡한 분산 JOIN이나 고카디널리티 GROUP BY에서 노드 간 데이터 교환이 필요하지만, 현재 지원하지 않는다.

**수십 노드까지는 괜찮으나 수백 노드에서는 문제**: 공식 문서가 직접 인정하는 확장성 한계. 동적으로 복제 영역을 분할하고 밸런싱하는 테이블 엔진이 필요하다고 명시.

공식 문서 원문:

> "This implementation gives you more control, and it is ok for relatively small clusters, such as tens of nodes. But for clusters with hundreds of nodes that we are using in production, this approach becomes a significant drawback."

### ClickHouse Cloud의 접근

ClickHouse Cloud에서는 이 한계를 다른 아키텍처로 해결한다:
- **SharedMergeTree**: 오브젝트 스토리지 기반으로 레플리카를 대체하여 고가용성 확보
- **Parallel Replicas**: 전통적 shared-nothing 클러스터의 shard와 유사하게 동작

---

## 8.5 ClickHouse Keeper

ZooKeeper의 ClickHouse 네이티브 대체품이다. 복제와 분산 DDL에 필수적이며, 복제 로그 저장, 리더 선출, 분산 DDL 조율을 담당한다.

Keeper가 필요한 경우:
- `ReplicatedMergeTree` 사용 시
- `ON CLUSTER` 분산 DDL 사용 시
- 분산 `INSERT ... SELECT` 사용 시

---

## 8.6 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Shard** | 데이터를 수평 분할하는 단위. 각 shard는 독립적 테이블 |
| **Distributed 테이블** | 데이터를 저장하지 않는 라우팅 레이어. SELECT 전달 + INSERT 라우팅 |
| **분산 SELECT** | 모든 shard에 쿼리 전달 → 로컬 실행 → Initiator에서 결과 병합 |
| **Global query plan 없음** | 1-pass 실행만 지원. 고카디널리티 GROUP BY, 대용량 JOIN에 한계 |
| **Replica** | shard 데이터의 복제본. 비동기 멀티마스터. conflict-free |
| **물리적 복제** | 쿼리가 아닌 압축 파트 전송. 머지는 각 레플리카에서 독립 수행 |
| **ZooKeeper/Keeper** | 복제 로그, 리더 선출, 일관성 복구의 중앙 조율 |
| **Not elastic** | 자동 리밸런싱 없음. 수십 노드까지 적합 |

---

## 8.7 운영 시 주의점

### Sharding Key 선택

- **`rand()`**: 균등 분산 보장. 대부분의 경우 적합. 단, 같은 엔티티의 데이터가 여러 shard에 분산되어 JOIN 비효율
- **결정적 키** (`cityHash64(user_id)` 등): 같은 사용자 데이터가 같은 shard에 모임. 특정 shard에 데이터 쏠림(skew) 위험
- **데이터 쏠림 모니터링**: shard 간 데이터 크기가 크게 차이나면 쿼리 성능이 불균일해진다

```sql
-- shard별 데이터 분포 확인
SELECT
    hostName() AS host,
    count() AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM clusterAllReplicas('test_cluster', system.parts)
WHERE database = 'uk' AND `table` = 'uk_price_paid_simple' AND active
GROUP BY host;
```

### INSERT Quorum

```sql
-- 기본: quorum 없음 → 노드 장애 시 방금 넣은 데이터 유실 가능
-- quorum 활성화: N개 레플리카에 확인 후 INSERT 완료
SET insert_quorum = 2;
INSERT INTO ...;
```

데이터 유실이 허용되지 않는 환경에서는 `insert_quorum` 설정을 활성화하라. 대신 INSERT 레이턴시가 증가한다.

### 복제 지연(Replication Lag) 모니터링

```sql
SELECT
    database,
    `table`,
    replica_name,
    is_leader,
    absolute_delay,
    queue_size,
    inserts_in_queue,
    merges_in_queue
FROM system.replicas
WHERE absolute_delay > 0
ORDER BY absolute_delay DESC;
```

`absolute_delay`가 증가 추세면 레플리카가 쓰기를 따라잡지 못하는 것이다. 네트워크 대역폭, 디스크 I/O, Keeper 상태를 확인하라.

### Distributed 테이블 INSERT 주의

Distributed 테이블에 INSERT하면 데이터가 Initiator 서버에 임시 저장된 후 비동기로 shard에 전달된다. Initiator 장애 시 이 임시 데이터가 유실될 수 있다.

**권장**: 가능하면 **로컬 테이블(shard)에 직접 INSERT**하라. 애플리케이션 레벨에서 sharding key를 계산하여 올바른 서버에 직접 전송하는 것이 가장 안전하다.

### Keeper 운영

- **홀수 노드** (3, 5, 7)로 Keeper 클러스터 구성 (과반수 합의를 위해)
- Keeper 장애 시 INSERT와 머지가 중단되지만, **SELECT는 정상 동작** (읽기는 Keeper에 의존하지 않음)
- Keeper 상태 모니터링: `system.zookeeper` 테이블, `four_letter_word_white_list` 커맨드

---

*다음 장: 9장~11장은 운영 실무 파트로, INSERT 전략, 피해야 할 패턴, 모니터링과 트러블슈팅을 다룬다.*

---

## 8.8 시각적 이해 — 분산 쿼리 라우팅 전 과정

### 클러스터 구성도

```
         ┌──────────────────────────────────────────────┐
         │              ClickHouse Keeper               │
         │          (3 노드 quorum, 복제 조율)            │
         └──────────────────────────────────────────────┘
                               ▲
                               │ (복제 로그, 리더 선출)
          ┌────────────────────┼────────────────────┐
          │                    │                    │
   ┌──────▼──────┐      ┌──────▼──────┐      ┌─────▼───────┐
   │   Shard 1   │      │   Shard 2   │      │   Shard 3   │
   │             │      │             │      │             │
   │ Replica A   │      │ Replica A   │      │ Replica A   │
   │ Replica B   │      │ Replica B   │      │ Replica B   │
   └─────────────┘      └─────────────┘      └─────────────┘
        ▲                    ▲                    ▲
        │                    │                    │
        └────────────────────┼────────────────────┘
                      Distributed 테이블
                             ▲
                             │
                         [클라이언트]
```

### SELECT 실행 흐름

```
① 클라이언트가 Distributed 테이블에 SELECT 전송
   SELECT count() FROM events_dist WHERE country = 'KR';
                 │
                 ▼
         Initiator 노드
                 │
② Distributed가 모든 shard에 쿼리 전달 (각 shard에서 1개 레플리카 선택)
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
Shard1-A    Shard2-B    Shard3-A   ← 병렬 로컬 실행
    │            │            │
    │ 로컬에서 count(*)         │
    │ 반환값: {count: 150}     │
    │            │            │
    └────────────┼────────────┘
                 ▼
③ Initiator가 결과 수집 및 병합
   {150} + {200} + {180} = {530}
                 │
                 ▼
④ 클라이언트에 최종 결과 반환
   count() = 530
```

### INSERT 라우팅

```sql
-- Distributed 테이블 정의
ENGINE = Distributed('cluster', 'db', 'events_local', cityHash64(user_id))
                                                      └── sharding key
```

```
INSERT INTO events_dist VALUES (user_id=1001, ...), (user_id=1002, ...);
                 │
                 ▼
         Initiator 노드
                 │
    각 행마다 sharding key 계산:
      cityHash64(1001) % 3 = 1 → Shard 2
      cityHash64(1002) % 3 = 0 → Shard 1
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
Shard1        Shard2        Shard3
 (1002)        (1001)
 INSERT        INSERT
```

---

## 8.9 시각적 이해 — 복제 동작

### 정상 케이스

```
시점 T0:
  Replica A: [파트1, 파트2]
  Replica B: [파트1, 파트2]  (동일)
  Replica C: [파트1, 파트2]

시점 T1 — Replica B에 INSERT:
  Replica A: [파트1, 파트2]
  Replica B: [파트1, 파트2, 파트3(NEW)]
  Replica C: [파트1, 파트2]
                    │
                    ▼
  Keeper에 "파트3 생성됨" 로그 기록

시점 T2 — A, C가 로그 감지 후 복사:
  Replica A: [파트1, 파트2, 파트3]  ← B로부터 파트 다운로드
  Replica B: [파트1, 파트2, 파트3]
  Replica C: [파트1, 파트2, 파트3]  ← B로부터 파트 다운로드
```

**쿼리가 아닌 파트 자체가 전송됨 (물리적 복제).**

### 머지 조율

```
시점 T3 — Replica A에서 머지 리더 선출:
  Keeper에 "파트1 + 파트2 → 머지" 로그 기록
                    │
                    ▼
  Replica A: 머지 실행 → 파트12 생성
  Replica B: 로그 읽고 동일 머지 독립 실행 → 파트12 생성
  Replica C: 로그 읽고 동일 머지 독립 실행 → 파트12 생성

  모든 레플리카가 byte-identical한 파트12 생성
  → 네트워크 전송 불필요
```

---

## 8.10 분산 쿼리에서 안티패턴

### Distributed 테이블에 INSERT (나쁨)

```
클라이언트 → Distributed(Initiator) 
               → 각 shard로 비동기 전송
               → Initiator 장애 시 임시 데이터 유실

권장:
클라이언트 → 직접 shard의 로컬 테이블에 INSERT
            (클라이언트가 sharding key 계산해서 라우팅)
```

### 고카디널리티 GROUP BY (나쁨)

```
SELECT user_id, sum(amount) FROM events_dist GROUP BY user_id;
                                               │
                         각 shard가 로컬 GROUP BY
                         → 수백만 개 유니크 키의 중간 결과 생성
                         → Initiator로 모두 전송 → 네트워크 폭증
                         → Initiator 메모리 부족

해결:
  1. sharding key를 user_id로 → 같은 유저 데이터가 한 shard에 모임
     → 각 shard에서 최종 집계 완료
  2. GROUP BY 후 LIMIT로 데이터 축소
  3. pre-aggregation (MV로 미리 집계)
```

### 전체 테이블 JOIN (나쁨)

```
SELECT * FROM events_dist e JOIN users_dist u ON e.user_id = u.id;
                                                 │
                         Initiator가 users 전체를 빌드 측으로
                         → 모든 shard에 브로드캐스트
                         → 큰 테이블이면 OOM

해결:
  1. 작은 테이블을 Dictionary로 변환 (인메모리 조인)
  2. GLOBAL JOIN 사용 (명시적 브로드캐스트)
  3. 같은 sharding key로 co-locate
```
