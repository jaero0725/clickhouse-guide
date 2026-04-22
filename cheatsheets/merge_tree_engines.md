# 치트시트: MergeTree 엔진 종류와 선택 기준

ClickHouse에는 MergeTree 패밀리의 다양한 엔진이 있다. 각 엔진은 머지 시점에 다른 일을 한다. **워크로드에 맞는 엔진을 선택하는 것이 성능의 시작**이다.

---

## 빠른 선택 플로우차트

```
데이터를 저장하려 함
  │
  ├── 단순 append-only (로그, 이벤트)?
  │     → MergeTree
  │
  ├── 같은 키의 최신 행만 필요 (유저 프로필)?
  │     → ReplacingMergeTree
  │
  ├── 같은 키의 숫자값을 합산 (일별 판매량)?
  │     → SummingMergeTree
  │
  ├── avg, uniq, quantile 같은 복잡한 집계?
  │     → AggregatingMergeTree + AggregateFunction
  │
  ├── 행의 "취소/상쇄" 로직이 필요 (부정 클릭)?
  │     → CollapsingMergeTree (순서 보장 필요)
  │     → VersionedCollapsingMergeTree (순서 미보장)
  │
  └── 정교한 그래프/트리 구조?
        → GraphiteMergeTree (Graphite 메트릭 특화)
```

---

## 전체 비교표

| 엔진 | 머지 시 동작 | 주 용도 | 주의사항 |
|------|-----------|--------|---------|
| **MergeTree** | 정렬 병합만 | 모든 append-only 데이터 | 가장 기본, 대부분 이걸로 시작 |
| **ReplacingMergeTree** | 같은 키 중 최신만 | 최신 상태 저장 (프로필) | 머지 전까지 중복 보임 → FINAL 필요 |
| **SummingMergeTree** | 숫자 컬럼 합산 | 단순 합계 대시보드 | 키에 없는 숫자만 합산됨 |
| **AggregatingMergeTree** | 부분 집계 상태 결합 | avg, uniq 등 복잡 집계 | AggregateFunction 타입 사용 |
| **CollapsingMergeTree** | sign +1/-1 상쇄 | 취소/롤백 로직 | 순서 꼬이면 collapse 안 됨 |
| **VersionedCollapsingMergeTree** | version + sign 상쇄 | 순서 미보장 환경 | version 관리 필요 |
| **ReplicatedMergeTree** | (위 모든 엔진의 복제판) | 클러스터 환경 | ZooKeeper/Keeper 필수 |

---

## 각 엔진 상세

### MergeTree — 기본

```sql
CREATE TABLE events (ts DateTime, user_id UInt64, event String)
ENGINE = MergeTree
ORDER BY (event, ts);
```

머지 시: 정렬 병합만 수행, 데이터 변환 없음.

---

### ReplacingMergeTree — 최신 상태

```sql
CREATE TABLE profile (
    user_id UInt64,
    grade   String,
    ver     UInt64   -- 버전 (선택)
)
ENGINE = ReplacingMergeTree(ver)   -- ver 최대값의 행이 살아남음
ORDER BY user_id;
```

머지 시:
```
같은 user_id 행들 중 ver 최대값만 유지
ver 생략 시 → 나중에 INSERT된 행이 살아남음
```

**쿼리 시 주의:**
```sql
-- 머지 전에는 중복 보임 → FINAL 또는 argMax
SELECT user_id, argMax(grade, ver) FROM profile GROUP BY user_id;
```

---

### SummingMergeTree — 자동 합산

```sql
CREATE TABLE sales_daily (
    date     Date,
    category String,
    quantity UInt32,  -- 자동 합산 대상
    amount   UInt64   -- 자동 합산 대상
)
ENGINE = SummingMergeTree
ORDER BY (category, date);
```

머지 시:
```
같은 (category, date) 행들 → 1개 행으로 병합
quantity, amount (ORDER BY에 없는 숫자) → 합산됨
```

**합산 대상 명시도 가능:**
```sql
ENGINE = SummingMergeTree((quantity, amount))
-- 괄호 안 컬럼만 합산, 다른 숫자 컬럼은 "임의의 값" 유지
```

---

### AggregatingMergeTree — 범용 집계

```sql
CREATE TABLE stats (
    date Date,
    user_id UInt64,
    avg_state  AggregateFunction(avg, Float32),
    uniq_state AggregateFunction(uniq, UInt64),
    p99_state  AggregateFunction(quantile(0.99), Float32)
)
ENGINE = AggregatingMergeTree
ORDER BY (date, user_id);

-- INSERT는 -State 함수로
INSERT INTO stats SELECT
    toDate(ts), user_id,
    avgState(value),
    uniqState(other_id),
    quantileState(0.99)(latency)
FROM raw_events GROUP BY ...;

-- SELECT는 -Merge 함수로
SELECT
    date,
    avgMerge(avg_state) AS avg_value,
    uniqMerge(uniq_state) AS unique_count,
    quantileMerge(0.99)(p99_state) AS p99
FROM stats
WHERE date = today()
GROUP BY date;
```

핵심: **부분 집계 상태를 저장**하므로 increment 가능.

---

### CollapsingMergeTree — 행 상쇄

```sql
CREATE TABLE orders (
    order_id UInt64,
    status   String,
    sign     Int8     -- +1 (기록), -1 (취소)
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY order_id;

-- 주문 생성
INSERT VALUES (1, 'pending', +1);

-- 주문 상태 변경 (이전 것 취소 + 새로 기록)
INSERT VALUES (1, 'pending', -1), (1, 'paid', +1);
```

머지 시:
```
같은 ORDER BY 키에서 +1, -1 쌍 발견 → 둘 다 삭제
최종적으로 최신 (1, 'paid', +1) 하나만 살아남음

주의: INSERT 순서가 중요. 순서 꼬이면 collapse 실패
      → VersionedCollapsingMergeTree 사용
```

---

### VersionedCollapsingMergeTree — 순서 독립 상쇄

```sql
ENGINE = VersionedCollapsingMergeTree(sign, version)

-- version이 같은 +1/-1끼리 상쇄 (순서 무관)
INSERT VALUES (1, 'pending', +1, 1);
INSERT VALUES (1, 'pending', -1, 1);  -- 언제 INSERT되든 상쇄
INSERT VALUES (1, 'paid', +1, 2);
```

---

## Replicated 버전

모든 엔진에 `Replicated` 접두사를 붙일 수 있음:

```sql
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/mytable', '{replica}')
ENGINE = ReplicatedReplacingMergeTree('/...', '{replica}', ver)
ENGINE = ReplicatedSummingMergeTree('/...', '{replica}')
ENGINE = ReplicatedAggregatingMergeTree('/...', '{replica}')
ENGINE = ReplicatedCollapsingMergeTree('/...', '{replica}', sign)
```

클러스터 환경에서는 반드시 Replicated 버전 사용.

---

## 함정들

### 1. SummingMergeTree는 key에 없는 숫자만 합산

```sql
-- 나쁜 예
CREATE TABLE t (
    date Date, user_id UInt64, amount UInt32
)
ENGINE = SummingMergeTree
ORDER BY (date, user_id, amount);  -- amount가 키에 있으면 합산 안 됨!

-- 좋은 예
ORDER BY (date, user_id)  -- amount는 키에 없음 → 합산됨
```

### 2. ReplacingMergeTree는 머지 전 중복 보임

```sql
-- 방금 INSERT 후 즉시 SELECT
SELECT * FROM profile WHERE user_id = 1;
-- 결과: 여러 행 (머지 전)

-- 해결 방법 3가지
SELECT * FROM profile FINAL WHERE user_id = 1;  -- 느림
SELECT argMax(grade, ver) FROM profile WHERE user_id = 1 GROUP BY user_id;  -- 권장
OPTIMIZE TABLE profile FINAL;  -- 무거움, 권장 X
```

### 3. AggregatingMergeTree의 State/Merge 혼동

```sql
-- INSERT 시: -State
avgState(value), uniqState(id), quantileState(0.99)(x)

-- SELECT 시: -Merge
avgMerge(...), uniqMerge(...), quantileMerge(0.99)(...)

-- If: -State 없이 그냥 avg → 오류
```

### 4. Collapsing 후 count() 잘못 사용

```sql
-- 나쁜 예 (+1, -1 행 모두 포함됨)
SELECT count() FROM orders WHERE status = 'paid';

-- 좋은 예
SELECT sum(sign) FROM orders WHERE status = 'paid';
```

---

## 실무 선택 가이드

| 워크로드 | 권장 엔진 |
|---------|---------|
| 일반 로그/이벤트 | MergeTree |
| 유저/상품 등 엔티티 최신 상태 | ReplacingMergeTree |
| 대시보드용 단순 합산 | SummingMergeTree + MV |
| 복잡한 집계 (avg, uniq, p99) | AggregatingMergeTree + MV |
| 주문/상태 변경이 잦음 | CollapsingMergeTree |
| 순서 보장 어려운 환경 | VersionedCollapsingMergeTree |
| 프로덕션 클러스터 | 위 모두의 Replicated 버전 |
