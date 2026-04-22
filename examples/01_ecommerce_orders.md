# 예제 1. E-commerce 주문/상품 분석

쇼핑몰의 주문 이벤트와 상품 클릭 로그를 분석하는 시나리오. 지금까지 배운 개념이 실무에 어떻게 적용되는지 끝까지 따라가 본다.

---

## 1. 요구사항

- 하루 주문 이벤트 약 1,000만 건
- 주요 분석 쿼리:
  - 특정 기간 GMV(총 매출)
  - 카테고리별 전환율
  - 유저별 최신 주문 상태 조회
  - 상품별 일별 판매량 추이
- 1년 지난 데이터는 자동 삭제

---

## 2. 테이블 설계

### 2.1 주문 이벤트 테이블 (주요 분석용)

```sql
CREATE TABLE shop.order_events (
    event_time    DateTime,
    order_id      UInt64,
    user_id       UInt64,
    product_id    UInt32,
    category_id   LowCardinality(String),   -- 'electronics', 'fashion' 등 수백 개
    channel       LowCardinality(String),   -- 'web', 'ios', 'android'
    amount        UInt32,
    quantity      UInt16,
    status        LowCardinality(String)    -- 'paid', 'refunded', 'canceled'
)
ENGINE = MergeTree
PARTITION BY toStartOfMonth(event_time)             -- 월 단위 (약 12개 파티션)
ORDER BY (category_id, toDate(event_time), user_id) -- 카테고리 → 날짜 → 유저
TTL event_time + INTERVAL 12 MONTH DELETE;
```

### 설계 근거

| 결정 | 이유 |
|------|------|
| `PARTITION BY toStartOfMonth` | TTL 삭제를 월 단위로 효율적으로 처리. 파티션 12개만 유지 |
| `ORDER BY (category_id, ...)` | category_id는 저카디널리티(수백) → 첫 번째 키로 대량 skip 가능 |
| `toDate(event_time)` | 날짜만 인덱스에 들어가도록 정밀도 축소 (8바이트→2바이트) |
| `LowCardinality(String)` | 수천 이하 고유값 컬럼에 적용 → 딕셔너리 인코딩으로 압축률·속도 향상 |
| `UInt32 amount` | 금액 상한이 42억이면 충분 (UInt64는 불필요) |

---

## 3. 유저 프로필 (최신 상태만 필요)

```sql
CREATE TABLE shop.user_profile (
    user_id       UInt64,
    grade         LowCardinality(String),   -- Bronze, Silver, Gold
    total_amount  UInt64,
    last_order_at DateTime,
    updated_at    DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;
```

유저 등급이 바뀌면 UPDATE가 아닌 새 행 INSERT:

```sql
-- UPDATE 대신 최신 상태로 INSERT
INSERT INTO shop.user_profile VALUES
  (123, 'Gold', 1500000, now(), now());

-- 조회: argMax로 최신 상태만
SELECT user_id, argMax(grade, updated_at) AS current_grade
FROM shop.user_profile
WHERE user_id = 123
GROUP BY user_id;
```

---

## 4. 일별 집계 테이블 (SummingMergeTree)

자주 보는 "카테고리 × 날짜 × 판매량" 대시보드는 매번 원본을 집계하면 느리다. SummingMergeTree로 사전 집계.

```sql
CREATE TABLE shop.sales_daily (
    date         Date,
    category_id  LowCardinality(String),
    quantity     UInt64,
    amount       UInt64,
    order_count  UInt64
)
ENGINE = SummingMergeTree
ORDER BY (category_id, date);

-- Materialized View로 실시간 자동 집계
CREATE MATERIALIZED VIEW shop.sales_daily_mv TO shop.sales_daily AS
SELECT
    toDate(event_time) AS date,
    category_id,
    sum(quantity)      AS quantity,
    sum(amount)        AS amount,
    count()            AS order_count
FROM shop.order_events
WHERE status = 'paid'
GROUP BY date, category_id;
```

```
INSERT order_events → MV가 자동으로 sales_daily에 집계 INSERT
         ↓ 백그라운드 머지
같은 (category_id, date) 행끼리 숫자 컬럼 합산 → 1개 행으로 수렴
```

---

## 5. INSERT 전략

### 5.1 배칭이 불가능한 경우 (웹 서버에서 주문 이벤트 발생)

```
문제: 서비스가 100개 pod에서 초당 1만 건씩 발생
  → 건 단위 INSERT 하면 초당 1만 파트 → 즉시 "Too many parts"
```

**해결 — Async Insert:**

```sql
-- 유저/프로파일 레벨로 활성화
ALTER USER app_writer SETTINGS 
    async_insert = 1,
    wait_for_async_insert = 1;
```

```
애플리케이션 pod 100개
         │ (각자 INSERT)
         ▼
서버 메모리 버퍼에 누적
         │
         ▼ [1MB 초과 or 1초 경과]
         │
파트 1개로 flush
```

### 5.2 배치 파이프라인 (Kafka Consumer)

```
Kafka → Consumer → 1만 건 누적 → INSERT
                     ↓
                   동기 INSERT, Native 포맷, LZ4 압축
```

```python
# 의사코드
buffer = []
for msg in kafka_consumer:
    buffer.append(msg)
    if len(buffer) >= 10000 or time_since_last_insert > 1:
        clickhouse.insert("shop.order_events", buffer, format="Native")
        buffer = []
```

---

## 6. 쿼리 패턴과 성능 분석

### 쿼리 1: 특정 카테고리 월 매출

```sql
SELECT sum(amount) AS gmv
FROM shop.order_events
WHERE category_id = 'electronics'
  AND event_time >= '2024-04-01'
  AND event_time <  '2024-05-01'
  AND status = 'paid';
```

**실행 흐름:**

```
① PARTITION 프루닝
   → 2024-04 파티션 1개만 스캔 (나머지 11개 파티션 skip)

② Primary Key 프루닝 (category_id, date)
   → electronics 영역의 granule만 남김
   → 전체 데이터의 약 1~2%만 실제 I/O

③ WHERE status = 'paid' 필터링 (벡터화)

④ sum(amount) 집계
```

`EXPLAIN indexes = 1` 결과 예:

```
MinMax:     Parts: 1/12,    Granules: 128/3800
Partition:  Parts: 1/1
PrimaryKey: Parts: 1/1,     Granules: 37/128
```

### 쿼리 2: 유저 최신 주문 상태 (잘못된 예)

```sql
-- 나쁜 예: user_id로 필터링
SELECT * FROM shop.order_events
WHERE user_id = 12345
ORDER BY event_time DESC LIMIT 10;
```

ORDER BY 키의 3번째가 user_id라서 **앞의 키(category_id)를 건너뛴 필터** → 인덱스가 먹히지 않음.

**개선안 1** — Projection 추가:

```sql
ALTER TABLE shop.order_events
ADD PROJECTION p_by_user (
    SELECT * ORDER BY (user_id, event_time)
);
```

**개선안 2** — 유저 기준 별도 테이블:

```sql
CREATE TABLE shop.order_events_by_user AS shop.order_events
ENGINE = MergeTree
PARTITION BY toStartOfMonth(event_time)
ORDER BY (user_id, event_time);

-- MV로 자동 복사
CREATE MATERIALIZED VIEW shop.mv_by_user
TO shop.order_events_by_user AS
SELECT * FROM shop.order_events;
```

---

## 7. 안티패턴 피하기

### 잘못된 예 1: 주문 상태를 UPDATE로 바꾸기

```sql
-- 금지
ALTER TABLE shop.order_events UPDATE status = 'refunded' 
WHERE order_id = 9999;
```

파트 전체를 다시 써야 함. 대신:

```sql
-- 이벤트를 append
INSERT INTO shop.order_events VALUES
  (now(), 9999, 123, ..., 'refunded');

-- 조회 시 최신 상태만
SELECT argMax(status, event_time) AS latest_status
FROM shop.order_events
WHERE order_id = 9999
GROUP BY order_id;
```

### 잘못된 예 2: 일 단위 파티셔닝

```sql
-- 금지 (3년이면 1,095개 파티션)
PARTITION BY toDate(event_time)
```

월 단위로 충분. 파티션 간 머지가 안 되므로 파티션이 많을수록 파트가 분산되어 관리가 어려워진다.

### 잘못된 예 3: 고카디널리티 컬럼을 ORDER BY 첫 번째로

```sql
-- 금지
ORDER BY (user_id, date)        -- user_id 수백만 고유값

-- 좋음
ORDER BY (category_id, date, user_id)
```

---

## 8. 운영 체크리스트

```sql
-- 파티션당 파트 수 (10 이하 유지)
SELECT partition, count() AS parts
FROM system.parts
WHERE database = 'shop' AND table = 'order_events' AND active
GROUP BY partition
ORDER BY parts DESC;

-- MV 지연 확인
SELECT 
    max(event_time) AS latest_raw,
    (SELECT max(date) FROM shop.sales_daily) AS latest_agg,
    latest_raw - latest_agg AS delay_sec
FROM shop.order_events;

-- 오래된 파티션 DROP (TTL 자동화 보완)
ALTER TABLE shop.order_events DROP PARTITION '2023-01-01';
```

---

## 9. 요약

| 기능 | 엔진 선택 | 이유 |
|------|----------|------|
| 주문 이벤트 (append-only) | MergeTree | 가장 기본, 변경 없음 |
| 유저 프로필 (상태 변경 있음) | ReplacingMergeTree | UPDATE 대신 신규 INSERT |
| 판매 집계 대시보드 | SummingMergeTree + MV | 자동 집계로 쿼리 가속 |
| 복잡한 통계(uniq, avg) | AggregatingMergeTree | 부분 집계 상태 저장 |
