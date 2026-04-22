# 예제 3. IoT 센서 데이터 — 시계열 분석

수만 개 IoT 기기에서 초당 한 번씩 메트릭을 전송하는 시나리오. 시계열 데이터의 ClickHouse 모델링.

---

## 1. 요구사항

- 100,000개 기기, 각 기기가 10초마다 측정값 전송
- 초당 약 10,000건 수집
- 쿼리 패턴:
  - 특정 기기의 시간대별 값 추이
  - 지역별 평균 온도/습도
  - 이상값(anomaly) 감지
- 3개월 지난 원본 데이터는 삭제, 일별 집계는 2년 보관

---

## 2. 원본 테이블 (Raw)

```sql
CREATE TABLE iot.sensor_raw (
    device_id   UInt32,
    region      LowCardinality(String),   -- 'seoul', 'busan', ... 수십 개
    sensor_type LowCardinality(String),   -- 'temp', 'humidity', 'pressure'
    ts          DateTime CODEC(DoubleDelta, LZ4),  -- 시계열 특화 코덱
    value       Float32  CODEC(Gorilla, LZ4)       -- 실수 시계열 특화
)
ENGINE = MergeTree
PARTITION BY toStartOfMonth(ts)
ORDER BY (region, sensor_type, device_id, ts)
TTL ts + INTERVAL 90 DAY DELETE;
```

### 코덱 선택이 중요한 이유

```
DateTime 컬럼 (ts):
  일반 ZSTD → 약 40% 압축
  DoubleDelta + LZ4 → 10% 이하까지 압축 (10배 차이!)
  이유: 연속된 타임스탬프가 일정한 간격 (10초)
        → DoubleDelta가 차이값의 차이값만 저장

Float32 컬럼 (value):
  일반 LZ4 → 센서값은 대부분 비슷 → 압축 어려움
  Gorilla + LZ4 → XOR 기반 압축 → 매우 효과적
  Facebook Gorilla 논문 기반, 시계열 특화
```

---

## 3. 집계 테이블 (분/시간/일 단위)

### 분 단위 집계 (1달 보관)

```sql
CREATE TABLE iot.sensor_1m (
    ts          DateTime,
    region      LowCardinality(String),
    sensor_type LowCardinality(String),
    device_id   UInt32,
    avg_state   AggregateFunction(avg, Float32),
    min_state   AggregateFunction(min, Float32),
    max_state   AggregateFunction(max, Float32),
    count_state AggregateFunction(count)
)
ENGINE = AggregatingMergeTree
PARTITION BY toStartOfWeek(ts)
ORDER BY (region, sensor_type, device_id, ts)
TTL ts + INTERVAL 30 DAY;
```

### Materialized View로 자동 집계

```sql
CREATE MATERIALIZED VIEW iot.sensor_1m_mv TO iot.sensor_1m AS
SELECT
    toStartOfMinute(ts) AS ts,
    region,
    sensor_type,
    device_id,
    avgState(value)   AS avg_state,
    minState(value)   AS min_state,
    maxState(value)   AS max_state,
    countState()      AS count_state
FROM iot.sensor_raw
GROUP BY ts, region, sensor_type, device_id;
```

### 집계 조회

```sql
-- 분 단위 평균/최소/최대 조회
SELECT
    ts,
    avgMerge(avg_state) AS avg_value,
    minMerge(min_state) AS min_value,
    maxMerge(max_state) AS max_value
FROM iot.sensor_1m
WHERE region = 'seoul'
  AND sensor_type = 'temp'
  AND device_id = 42
  AND ts >= now() - INTERVAL 1 HOUR
GROUP BY ts
ORDER BY ts;
```

**핵심 개념 — AggregateFunction의 부분 집계 상태**

```
avg 계산은 단순 합산이 안 됨:
  ❌ avg(avg(x), avg(y)) ≠ avg(x, y)

  But AggregatingMergeTree의 avgState는:
  state = {sum, count} 를 저장
  
  머지 시:
  {sum=10, count=5} + {sum=15, count=10} = {sum=25, count=15}
                                            → avg = 25/15
  정확함!
```

---

## 4. 일 단위 집계 (2년 보관)

```sql
CREATE TABLE iot.sensor_1d (
    date        Date,
    region      LowCardinality(String),
    sensor_type LowCardinality(String),
    device_id   UInt32,
    avg_state   AggregateFunction(avg, Float32),
    p95_state   AggregateFunction(quantile(0.95), Float32),
    p99_state   AggregateFunction(quantile(0.99), Float32)
)
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (region, sensor_type, device_id, date)
TTL date + INTERVAL 730 DAY;

-- 1m 테이블에서 다시 일 단위로 re-aggregate
CREATE MATERIALIZED VIEW iot.sensor_1d_mv TO iot.sensor_1d AS
SELECT
    toDate(ts) AS date,
    region, sensor_type, device_id,
    avgState(value) AS avg_state,
    quantileState(0.95)(value) AS p95_state,
    quantileState(0.99)(value) AS p99_state
FROM iot.sensor_raw
GROUP BY date, region, sensor_type, device_id;
```

---

## 5. 데이터 Lifecycle 시각화

```
[초당 10,000건 INSERT]
          │
          ▼
┌──────────────────────┐
│  sensor_raw (90일)   │ ← 원본 그대로
└──────────┬───────────┘
           │ MV
           ▼
┌──────────────────────┐
│  sensor_1m (30일)    │ ← 분 단위 집계
└──────────┬───────────┘
           │ MV
           ▼
┌──────────────────────┐
│  sensor_1d (2년)     │ ← 일 단위 집계
└──────────────────────┘

최근 쿼리 → sensor_raw (분 이하 정밀도)
중기 쿼리 → sensor_1m  (분 단위)
장기 쿼리 → sensor_1d  (일 단위)
```

---

## 6. 시계열 특화 함수

### 갭 보간 (Gap Filling)

```sql
-- 10초 간격이지만 누락된 구간이 있을 때 채우기
SELECT
    toStartOfInterval(ts, INTERVAL 10 SECOND) AS bucket,
    avg(value) AS avg_value
FROM iot.sensor_raw
WHERE device_id = 42
  AND ts >= now() - INTERVAL 10 MINUTE
GROUP BY bucket WITH FILL
  FROM now() - INTERVAL 10 MINUTE
  TO now()
  STEP INTERVAL 10 SECOND;
```

`WITH FILL`은 누락된 구간을 NULL로 채움 → Grafana 같은 도구에서 끊김 없이 표시.

### 이전 값과 비교 (Window Function)

```sql
-- 급격한 변화 감지
SELECT
    ts, device_id, value,
    value - lagInFrame(value) OVER w AS delta
FROM iot.sensor_raw
WHERE device_id = 42
  AND ts >= now() - INTERVAL 1 HOUR
WINDOW w AS (PARTITION BY device_id ORDER BY ts);
```

---

## 7. 이상값 감지 쿼리

```sql
-- 지난 24시간 평균 대비 3시그마 이탈한 기기 찾기
WITH baseline AS (
    SELECT
        device_id,
        avg(value) AS mu,
        stddevPop(value) AS sigma
    FROM iot.sensor_raw
    WHERE sensor_type = 'temp'
      AND ts >= now() - INTERVAL 24 HOUR
    GROUP BY device_id
)
SELECT
    r.device_id,
    r.value,
    b.mu AS baseline_avg,
    (r.value - b.mu) / b.sigma AS z_score
FROM iot.sensor_raw AS r
INNER JOIN baseline AS b ON r.device_id = b.device_id
WHERE r.ts >= now() - INTERVAL 5 MINUTE
  AND abs((r.value - b.mu) / b.sigma) > 3;
```

---

## 8. 흔한 실수

| 실수 | 문제 | 해결 |
|------|------|------|
| 코덱 미설정 | 디스크 사용량 5~10배 | DoubleDelta, Gorilla 적용 |
| 원본 데이터에 직접 대시보드 쿼리 | 항상 풀 스캔 | Materialized View로 집계 |
| 집계 테이블에 SummingMergeTree 사용 + avg | 부정확 | AggregatingMergeTree + avgState |
| PARTITION BY ts (초 단위) | 파티션 무한 증식 | Month/Week 단위 |
| Float64 기본 사용 | 용량 2배 | 센서값은 보통 Float32면 충분 |

---

## 9. 운영 체크리스트

```sql
-- 테이블별 압축률 확인
SELECT
    name,
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.parts
WHERE database = 'iot' AND active
GROUP BY name
ORDER BY compressed DESC;

-- MV 동작 확인 (raw vs 1m 최신 시각 비교)
SELECT
    (SELECT max(ts) FROM iot.sensor_raw) AS raw_latest,
    (SELECT max(ts) FROM iot.sensor_1m) AS m1_latest,
    raw_latest - m1_latest AS lag_seconds;
```
