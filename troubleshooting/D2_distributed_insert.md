# D2. Distributed 테이블에 INSERT — 이중 네트워크 홉

## 증상

- INSERT 레이턴시가 로컬 테이블보다 2~5배 높음
- 네트워크 트래픽이 예상의 2배
- 특정 노드에 INSERT 부하가 집중
- 간헐적 INSERT 실패 (네트워크 일시 장애 시)

---

## 원인

Distributed 테이블은 **라우팅 레이어**다. 데이터를 직접 저장하지 않고 로컬 테이블로 전달한다.

```
Distributed 테이블에 INSERT 시 실제 흐름:

  애플리케이션
      │
      ▼
  node1 (Distributed 수신)
      │
      ├──→ 디스크에 임시 버퍼 파일 저장
      │
      ├──→ node1 로컬 테이블 (직접 전달)
      ├──→ node2 로컬 테이블 (네트워크 전송)
      └──→ node3 로컬 테이블 (네트워크 전송)

  → 데이터가 네트워크를 2번 탄다:
    1. 애플리케이션 → node1 (수신)
    2. node1 → node2, node3 (샤드로 전달)

  → 임시 버퍼 파일이 디스크에 쌓임
  → node1에 라우팅 부하 집중
```

추가 문제:
- node1이 일시 장애 시 버퍼 파일이 남아 디스크 공간 낭비
- 버퍼 → 샤드 전송 실패 시 재시도 로직 복잡

---

## 진단

```sql
-- Distributed 테이블 INSERT 버퍼 확인
SELECT *
FROM system.distributed_ddl_queue
WHERE entry_host = hostname();

-- 버퍼 파일 남아있는지 확인
-- (데이터 디렉토리의 .bin 파일들)
-- 경로: /var/lib/clickhouse/data/{db}/{distributed_table}/

-- INSERT 레이턴시 비교
-- 직접 로컬 테이블 INSERT
INSERT INTO events_local VALUES (...);    -- 측정

-- Distributed 테이블 INSERT
INSERT INTO events_dist VALUES (...);     -- 측정
-- 차이가 2배 이상이면 Distributed INSERT 병목
```

---

## 해결

### 핵심 원칙: 쓰기는 로컬, 읽기는 Distributed

```sql
-- 테이블 구조 (올바른 역할 분리)

-- 1. 로컬 테이블 (실제 데이터 저장)
CREATE TABLE events_local ON CLUSTER my_cluster (
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY (user_id, event_time);

-- 2. Distributed 테이블 (읽기 전용 라우터)
CREATE TABLE events ON CLUSTER my_cluster
AS events_local
ENGINE = Distributed(my_cluster, currentDatabase(), events_local, user_id);
```

```sql
-- 나쁜 예: Distributed 테이블에 INSERT
INSERT INTO events VALUES (...);            -- 이중 홉 발생

-- 좋은 예: 로컬 테이블에 직접 INSERT
INSERT INTO events_local VALUES (...);      -- 단일 홉
```

### 샤드별로 직접 INSERT하는 방법

```python
# 각 샤드의 노드에 직접 연결해 INSERT
import hashlib

SHARD_NODES = {
    0: "node1:9000",
    1: "node2:9000",
    2: "node3:9000",
}
SHARD_COUNT = len(SHARD_NODES)

def get_shard(user_id: int) -> int:
    return user_id % SHARD_COUNT

# user_id별로 올바른 샤드 노드에 직접 INSERT
for batch in batches:
    shard_batches = {}
    for row in batch:
        shard = get_shard(row['user_id'])
        shard_batches.setdefault(shard, []).append(row)

    for shard, rows in shard_batches.items():
        node = SHARD_NODES[shard]
        client = get_client(node)
        client.execute("INSERT INTO events_local VALUES", rows)
```

### Kafka를 통한 샤드 분산

```sql
-- 각 노드에 Kafka 엔진 테이블 생성 (각 샤드가 직접 Kafka 소비)
CREATE TABLE kafka_events ON CLUSTER my_cluster (
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'clickhouse_{shard}',  -- 샤드별 컨슈머 그룹
    kafka_format = 'JSONEachRow';

-- 로컬 테이블로 바로 적재 (Distributed 거치지 않음)
CREATE MATERIALIZED VIEW kafka_mv TO events_local AS
SELECT * FROM kafka_events;
```

---

## Distributed INSERT가 불가피한 경우

클라이언트가 샤딩 로직을 알 수 없는 경우 등.

```sql
-- 그나마 덜 나쁜 설정
SET insert_distributed_sync = 1;  -- 비동기 대신 동기 전송 (데이터 안전)

-- 버퍼 크기 조정 (임시 파일 생성 방지)
SET distributed_directory_monitor_sleep_time_ms = 100;
SET distributed_directory_monitor_max_sleep_time_ms = 5000;
```

---

## 예방

```
INSERT 경로 설계 원칙:

  ✅ 애플리케이션 → 로컬 테이블 (직접)
  ✅ Kafka/MQ → Kafka 엔진 → 로컬 테이블
  ✅ Distributed 테이블 = SELECT 전용

  ❌ 애플리케이션 → Distributed 테이블
  ❌ Materialized View → Distributed 테이블 (MV는 로컬 테이블로)

  이름 컨벤션으로 실수 방지:
    events_local  → 로컬 테이블 (INSERT 대상)
    events        → Distributed 테이블 (SELECT 대상)
```
