# 예제 4. 유저 행동 분석 (Product Analytics)

Amplitude, Mixpanel 같은 Product Analytics 시스템이 ClickHouse로 구현될 때의 패턴. Funnel, Retention, DAU/MAU 계산.

---

## 1. 요구사항

- 모바일/웹 앱 유저 이벤트 (view, click, sign_up, purchase 등)
- DAU 수천만 → 일 수억 건 이벤트
- 쿼리:
  - Funnel (단계별 전환율)
  - DAU/WAU/MAU (고유 유저 수)
  - Retention (유저 잔존율)
  - Cohort 분석

---

## 2. 테이블 설계

### 2.1 원본 이벤트 테이블

```sql
CREATE TABLE analytics.events (
    event_time DateTime,
    event_date Date MATERIALIZED toDate(event_time),
    event_name LowCardinality(String),
    user_id    UInt64,
    session_id UInt64,
    device     LowCardinality(String),
    os         LowCardinality(String),
    app_ver    LowCardinality(String),
    country    LowCardinality(String),
    properties Map(String, String)
)
ENGINE = MergeTree
PARTITION BY event_date
ORDER BY (event_name, event_date, user_id)
TTL event_time + INTERVAL 2 YEAR DELETE;
```

### 설계 포인트

- `event_date MATERIALIZED`: 자동 계산 컬럼, INSERT 시 계산
- `ORDER BY (event_name, event_date, user_id)`: 이벤트 타입 필터가 대부분 → 첫 번째 키
- `LowCardinality`: device/os/country는 수십~수백 → 딕셔너리 인코딩 대폭 절약

---

## 3. DAU/MAU 집계 (HyperLogLog)

정확한 고유 유저 카운팅은 비싸다 → HyperLogLog 근사로 빠르게.

```sql
CREATE TABLE analytics.dau (
    date         Date,
    country      LowCardinality(String),
    hll_state    AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY (country, date);

CREATE MATERIALIZED VIEW analytics.dau_mv TO analytics.dau AS
SELECT
    toDate(event_time) AS date,
    country,
    uniqState(user_id) AS hll_state
FROM analytics.events
GROUP BY date, country;
```

### 조회

```sql
-- DAU
SELECT date, uniqMerge(hll_state) AS dau
FROM analytics.dau
WHERE date >= today() - 30
GROUP BY date ORDER BY date;

-- MAU (30일 롤링)
SELECT uniqMerge(hll_state) AS mau
FROM analytics.dau
WHERE date >= today() - 30;

-- 국가별 DAU
SELECT country, uniqMerge(hll_state) AS dau
FROM analytics.dau
WHERE date = today()
GROUP BY country
ORDER BY dau DESC;
```

**HyperLogLog의 마법**:

```
정확한 uniq: 메모리에 모든 user_id 해시 저장 → 1억 유저 → 수 GB
HyperLogLog: 고정 크기 카운터 → 약 12KB로 1억 유저 추정
            오차 ~1% 수준, 속도는 수백 배 빠름

MV에 uniqState 저장 → 머지 시 HLL 카운터끼리 결합 (정확한 증분)
```

---

## 4. Funnel 분석

```sql
-- 3단계 퍼널: view → sign_up → purchase
SELECT
    level,
    count() AS users
FROM (
    SELECT
        user_id,
        windowFunnel(3600)(             -- 1시간 윈도우
            event_time,
            event_name = 'view',
            event_name = 'sign_up',
            event_name = 'purchase'
        ) AS level
    FROM analytics.events
    WHERE event_date = today()
      AND event_name IN ('view', 'sign_up', 'purchase')
    GROUP BY user_id
)
GROUP BY level
ORDER BY level;
```

**결과 해석:**

```
level=0: view도 안 함 (조건 자체 안 맞음)
level=1: view만 함
level=2: view → sign_up 까지
level=3: view → sign_up → purchase 완료

전환율:
  view → sign_up = count(level≥2) / count(level≥1)
  sign_up → purchase = count(level=3) / count(level≥2)
```

### windowFunnel 동작 원리

```
유저 123의 이벤트:
  09:00 view
  09:15 click
  09:30 sign_up      ← 조건 2 매칭
  10:00 view
  10:30 purchase     ← 조건 3 매칭

windowFunnel(3600): "1시간 내에 순서대로 매칭되는 단계 수" 반환
  시작: 09:00 view (조건 1)
  09:30 sign_up (조건 2) - 30분 경과, 1시간 이내 OK
  10:30 purchase (조건 3) - 09:00부터 1.5시간 → 윈도우 초과 X
  
→ 최장 연속 매칭 단계 반환
```

---

## 5. Retention (잔존율)

```sql
-- 가입일 기준 N일 후 복귀율
WITH signup_cohort AS (
    SELECT user_id, toDate(min(event_time)) AS signup_date
    FROM analytics.events
    WHERE event_name = 'sign_up'
      AND event_date >= today() - 30
    GROUP BY user_id
)
SELECT
    signup_date,
    count(DISTINCT user_id) AS cohort_size,
    uniqIf(user_id, event_date = signup_date + 1) AS day1_active,
    uniqIf(user_id, event_date = signup_date + 7) AS day7_active,
    uniqIf(user_id, event_date = signup_date + 30) AS day30_active,
    day1_active / cohort_size AS d1_retention,
    day7_active / cohort_size AS d7_retention,
    day30_active / cohort_size AS d30_retention
FROM analytics.events AS e
INNER JOIN signup_cohort AS s ON e.user_id = s.user_id
WHERE e.event_date BETWEEN s.signup_date AND s.signup_date + 30
GROUP BY signup_date
ORDER BY signup_date;
```

---

## 6. Session 분석

세션은 "30분 이상 끊기면 다른 세션"으로 정의되는 게 일반적. ClickHouse의 `arrayCumSum` + `neighbor`로 구현:

```sql
SELECT
    user_id,
    session_start,
    session_end,
    session_end - session_start AS duration_sec,
    event_count
FROM (
    SELECT
        user_id,
        groupArray(event_time) AS times,
        arrayMap((ts, i) -> 
            if(i = 1 OR (ts - times[i-1]) > 1800, 1, 0),
            times,
            arrayEnumerate(times)
        ) AS is_new_session,
        arrayCumSum(is_new_session) AS session_id
    FROM (
        SELECT user_id, event_time
        FROM analytics.events
        WHERE event_date = today()
        ORDER BY user_id, event_time
    )
    GROUP BY user_id
)
ARRAY JOIN 
    times AS event_time,
    session_id AS sid
GROUP BY user_id, sid
HAVING count() > 1;
```

---

## 7. 실시간 카운터 대시보드

"지금 실시간 활성 유저"는 최신 1분 데이터를 빠르게 보여줘야 함:

```sql
-- 5초마다 폴링하는 쿼리
SELECT 
    uniq(user_id) AS active_now,
    countIf(event_name = 'purchase') AS purchases_last_min
FROM analytics.events
WHERE event_time >= now() - INTERVAL 1 MINUTE;
```

**주의 — 방금 INSERT된 데이터는 안 보일 수 있음:**

```
INSERT 후 파트 생성은 즉시
하지만 Async Insert 사용 중이면 flush까지 최대 1초 지연
→ 완전 실시간이 필요하면 wait_for_async_insert=1로 flush 기다리기
```

---

## 8. A/B 테스트 분석

```sql
CREATE TABLE analytics.experiments (
    user_id      UInt64,
    experiment   LowCardinality(String),
    variant      LowCardinality(String),       -- 'control', 'treatment'
    assigned_at  DateTime
)
ENGINE = ReplacingMergeTree(assigned_at)
ORDER BY (experiment, user_id);

-- 실험군별 전환율 비교
SELECT
    variant,
    uniq(user_id) AS users,
    countIf(e.event_name = 'purchase') AS purchases,
    purchases / users AS conversion_rate
FROM analytics.experiments AS x
INNER JOIN analytics.events AS e ON x.user_id = e.user_id
WHERE x.experiment = 'checkout_redesign_v2'
  AND e.event_date >= '2024-04-01'
GROUP BY variant;
```

---

## 9. 흔한 실수

| 실수 | 문제 | 해결 |
|------|------|------|
| `DISTINCT user_id` 사용 | 1억 유저에서 수십 GB 메모리 | `uniq` (HyperLogLog) |
| ORDER BY에 user_id 첫 번째 | 이벤트 타입 필터 안 먹힘 | event_name을 앞에 |
| Funnel을 JOIN으로 구현 | 수시간 걸림 | windowFunnel 함수 사용 |
| 모든 쿼리가 원본 테이블 | 대시보드 느림 | MV로 사전 집계 |
| user_id를 String으로 | 정수 대비 3~5배 느림 | UInt64로 |

---

## 10. 성능 비교

동일 DAU 쿼리 (1일, 500만 유저):

| 방식 | 소요 시간 | 메모리 |
|------|---------|--------|
| `SELECT count(DISTINCT user_id)` 원본 | 8,500ms | 2.1 GB |
| `SELECT uniq(user_id)` 원본 | 1,200ms | 130 MB |
| `uniqMerge` MV 테이블 | 12ms | 8 MB |

**MV 도입만으로 700배 빨라짐.**
