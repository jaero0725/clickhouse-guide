# C3. Distributed 테이블 — 고카디널리티 GROUP BY로 OOM

## 증상

```
DB::Exception: Memory limit (for query) exceeded: would use 32.00 GiB
```

- 분산 테이블 조회 시 특정 노드만 메모리 폭발
- 샤드를 추가해도 쿼리가 느려지거나 OOM 발생
- 로컬 테이블로 같은 쿼리 실행 시 문제 없음

---

## 원인

Distributed 테이블 쿼리는 **2단계 집계**로 실행된다.

```
클러스터 3샤드 구성:
  Shard1 (node1): user_id 1~33만
  Shard2 (node2): user_id 34~66만
  Shard3 (node3): user_id 67~100만

SELECT user_id, sum(amount) FROM dist_table GROUP BY user_id;

1단계 — 각 샤드에서 로컬 집계:
  node1: 33만 user_id × 집계 → 33만 행 반환
  node2: 33만 user_id × 집계 → 33만 행 반환
  node3: 33만 user_id × 집계 → 33만 행 반환

2단계 — 이니시에이터(코디네이터) 노드에서 최종 집계:
  33만 + 33만 + 33만 = 99만 행을 메모리에 올려 GROUP BY
  → user_id가 샤드 간에 분산되어 있어 메모리에서 모두 합쳐야 함
```

```
문제: user_id가 100만 명 × 행당 ~200바이트 = 200MB
     실제로는 중간 집계 상태, 정렬 버퍼 등 포함 시 수 GB
     샤드가 많을수록 이니시에이터 노드에 모이는 데이터가 더 많아짐
```

---

## 진단

```sql
-- 쿼리 실행 시 노드별 메모리 사용 확인
SELECT
    hostname(),
    query_id,
    peak_memory_usage,
    query
FROM clusterAllReplicas('cluster_name', system.query_log)
WHERE type = 'QueryFinish'
  AND query_start_time > now() - INTERVAL 1 HOUR
ORDER BY peak_memory_usage DESC
LIMIT 20;

-- GROUP BY 키 카디널리티 확인
SELECT uniqExact(user_id) FROM dist_table;
-- 결과: 1,234,567 → 고카디널리티, 분산 GROUP BY 위험
```

---

## 해결

### 해결책 1: GROUP BY 키를 샤딩 키와 일치시키기

분산 테이블이 user_id로 샤딩되어 있다면, user_id GROUP BY는 각 샤드에서 완전히 처리된다.

```sql
-- 현재 샤딩 키 확인
SELECT sharding_key FROM system.tables WHERE name = 'dist_table';

-- 이상적인 경우: 샤딩 키 = GROUP BY 키
-- dist_table이 user_id로 샤딩되어 있다면
SELECT user_id, sum(amount)
FROM dist_table
GROUP BY user_id;
-- → 각 샤드에서 완결, 이니시에이터는 합산만

-- 재설계가 필요하다면 샤딩 키를 쿼리 패턴에 맞게 변경
```

### 해결책 2: 메모리 제한 완화 (임시 방편)

```sql
-- 쿼리별 메모리 한도 설정
SET max_memory_usage = 64000000000;  -- 64GB

-- 또는 외부 집계 허용 (메모리 부족 시 디스크 사용)
SET group_by_two_level_threshold = 100000;
SET group_by_two_level_threshold_bytes = 50000000;
SET max_bytes_before_external_group_by = 20000000000;  -- 20GB

SELECT user_id, sum(amount) FROM dist_table GROUP BY user_id;
```

### 해결책 3: 정확도 희생으로 메모리 절감 (카운팅)

```sql
-- user_id 고유 카운트: DISTINCT 대신 uniq (HyperLogLog)
-- 나쁜 예
SELECT count(DISTINCT user_id) FROM dist_table;  -- 이니시에이터에 전체 user_id 집결

-- 좋은 예 (메모리 100배 절감, 오차 ~1%)
SELECT uniq(user_id) FROM dist_table;
```

### 해결책 4: Materialized View로 사전 집계

고카디널리티 GROUP BY가 자주 쓰이는 쿼리라면 사전 집계가 근본 해결책.

```sql
-- 일별 유저 집계 사전 계산
CREATE TABLE user_daily_agg_local ON CLUSTER cluster_name (
    date        Date,
    user_id     UInt64,
    total_amount UInt64,
    order_count  UInt32
)
ENGINE = ReplicatedSummingMergeTree('/clickhouse/tables/{shard}/user_daily', '{replica}')
ORDER BY (user_id, date);

CREATE TABLE user_daily_agg ON CLUSTER cluster_name
AS user_daily_agg_local
ENGINE = Distributed(cluster_name, currentDatabase(), user_daily_agg_local, user_id);

CREATE MATERIALIZED VIEW user_daily_mv ON CLUSTER cluster_name
TO user_daily_agg_local AS
SELECT
    toDate(event_time) AS date,
    user_id,
    sum(amount)        AS total_amount,
    count()            AS order_count
FROM events_local
GROUP BY date, user_id;

-- 이후 조회는 user_daily_agg에서 (이미 집계됨)
SELECT user_id, sum(total_amount)
FROM user_daily_agg
WHERE date >= today() - 30
GROUP BY user_id;
```

### 해결책 5: distributed_group_by_no_merge 설정

샤딩 키와 GROUP BY 키가 일치할 때 이니시에이터의 재집계를 건너뛰는 설정.

```sql
SET distributed_group_by_no_merge = 1;
-- GROUP BY 결과를 이니시에이터에서 재집계하지 않고 바로 반환
-- 주의: 샤딩 키와 GROUP BY 키가 일치할 때만 사용 (결과 틀릴 수 있음)
```

---

## 예방

```
분산 쿼리 설계 원칙:

  ✅ 샤딩 키 = 자주 쓰이는 GROUP BY 키
  ✅ 고카디널리티 집계는 사전 MV로 처리
  ✅ 카운팅은 DISTINCT 대신 uniq/uniqHLL12
  ✅ 분산 쿼리 전 로컬에서 메모리 사용량 추정

  추정 공식:
  이니시에이터 메모리 ≈ GROUP BY 고유값 수 × 집계 컬럼 크기 × 샤드 수 × 2~3배
```
