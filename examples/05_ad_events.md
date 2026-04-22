# 예제 5. 광고 이벤트 — impression/click/conversion

광고 플랫폼의 이벤트 수집과 실시간 과금 계산. CollapsingMergeTree와 정밀한 카운팅이 중요한 도메인.

---

## 1. 요구사항

- impression: 초당 100만 건 (대부분의 이벤트)
- click: 초당 10만 건
- conversion: 초당 1천 건
- 실시간 과금, CTR 대시보드, advertiser별 리포트
- 부정 클릭 취소 (invalidate)

---

## 2. 이벤트 테이블

```sql
CREATE TABLE ads.events (
    event_time  DateTime,
    event_type  LowCardinality(String),  -- 'impression', 'click', 'conversion'
    ad_id       UInt64,
    campaign_id UInt32,
    advertiser  LowCardinality(String),
    user_id     UInt64,
    device      LowCardinality(String),
    country     LowCardinality(String),
    price       UInt32,                  -- 센트 단위 (정수)
    
    -- CollapsingMergeTree용
    sign        Int8
)
ENGINE = CollapsingMergeTree(sign)
PARTITION BY toYYYYMM(event_time)
ORDER BY (advertiser, event_type, toDate(event_time), ad_id);
```

---

## 3. CollapsingMergeTree 동작 원리

부정 클릭을 "취소"하기 위해 CollapsingMergeTree 사용:

```
클릭 이벤트 기록:
  INSERT VALUES (..., ad_id=1, price=100, sign=+1)

부정 클릭 판정 → 취소:
  INSERT VALUES (..., ad_id=1, price=100, sign=-1)
  (기존과 모든 값 동일, sign만 -1)

백그라운드 머지 시:
  ┌────────────────────────────────────────┐
  │ (..., ad_id=1, price=100, sign=+1)     │
  │ (..., ad_id=1, price=100, sign=-1)     │
  │                ↓ 같은 키, 부호 반대     │
  │        서로 상쇄 (collapse)             │
  │                ↓                        │
  │               삭제                      │
  └────────────────────────────────────────┘
```

### 합산 쿼리

```sql
-- 유효 클릭 수 (부정 제외)
SELECT
    advertiser,
    sum(sign) AS valid_clicks,        -- sign을 합산하면 +1/-1이 상쇄
    sum(price * sign) AS total_cost
FROM ads.events
WHERE event_type = 'click'
  AND event_time >= today()
GROUP BY advertiser;
```

주의 — **머지 전**에는 +1/-1 행이 둘 다 존재하므로 반드시 `sum(sign)` 패턴 사용. 단순 `count()`는 잘못된 결과.

---

## 4. 실시간 집계 (SummingMergeTree)

CTR 대시보드는 SummingMergeTree로 사전 집계:

```sql
CREATE TABLE ads.stats_hourly (
    hour        DateTime,
    advertiser  LowCardinality(String),
    campaign_id UInt32,
    impressions UInt64,
    clicks      UInt64,
    conversions UInt64,
    spend       UInt64
)
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(hour)
ORDER BY (advertiser, campaign_id, hour);

CREATE MATERIALIZED VIEW ads.stats_hourly_mv TO ads.stats_hourly AS
SELECT
    toStartOfHour(event_time) AS hour,
    advertiser,
    campaign_id,
    sumIf(sign, event_type = 'impression')    AS impressions,
    sumIf(sign, event_type = 'click')         AS clicks,
    sumIf(sign, event_type = 'conversion')    AS conversions,
    sumIf(price * sign, event_type = 'click') AS spend
FROM ads.events
GROUP BY hour, advertiser, campaign_id;
```

### CTR 조회

```sql
SELECT
    campaign_id,
    sum(impressions) AS impr,
    sum(clicks) AS clk,
    sum(clicks) / sum(impressions) AS ctr,
    sum(spend) / 100 AS spend_dollars      -- 센트 → 달러
FROM ads.stats_hourly
WHERE advertiser = 'Nike'
  AND hour >= today()
GROUP BY campaign_id
ORDER BY spend_dollars DESC;
```

---

## 5. 부정 클릭 탐지 패턴

동일 IP에서 단시간 내 과다 클릭:

```sql
SELECT
    device,
    user_id,
    count() AS click_count,
    min(event_time) AS first,
    max(event_time) AS last,
    (max(event_time) - min(event_time)) AS span_sec
FROM ads.events
WHERE event_type = 'click'
  AND sign = 1
  AND event_time >= now() - INTERVAL 5 MINUTE
GROUP BY device, user_id
HAVING click_count > 10 AND span_sec < 60;
```

탐지된 경우 cancel INSERT:

```sql
INSERT INTO ads.events
SELECT event_time, event_type, ad_id, campaign_id, advertiser,
       user_id, device, country, price, -1 AS sign
FROM ads.events
WHERE user_id = <fraudulent_user>
  AND event_type = 'click'
  AND sign = 1;
```

---

## 6. VersionedCollapsingMergeTree (순서가 중요할 때)

CollapsingMergeTree는 INSERT 순서가 보장되어야 작동. 순서가 꼬이면 collapse가 안 될 수 있음. 안정적인 대안:

```sql
CREATE TABLE ads.events_v2 (
    ...,
    sign    Int8,
    version UInt64
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY (...);

-- 원본: version=1, sign=+1
INSERT VALUES (..., 1, +1);

-- 취소: 같은 version, sign=-1
INSERT VALUES (..., 1, -1);
```

순서가 섞여도 version 매칭으로 정확히 collapse.

---

## 7. 분산 샤딩

수백 광고주 × 초당 100만 건 → 단일 서버 한계:

```sql
-- Shard 1~4에 ReplicatedMergeTree + 각 shard에 2개 replica
CREATE TABLE ads.events_local ON CLUSTER ads_cluster (...)
ENGINE = ReplicatedCollapsingMergeTree(...)
ORDER BY (advertiser, event_type, toDate(event_time), ad_id);

-- Distributed 테이블 (read용, 샤딩 키는 advertiser 해시)
CREATE TABLE ads.events ON CLUSTER ads_cluster AS ads.events_local
ENGINE = Distributed('ads_cluster', 'ads', 'events_local',
                     cityHash64(advertiser));
```

**Sharding Key가 `advertiser` 해시인 이유:**

```
같은 advertiser 데이터가 같은 shard에 집중
  → GROUP BY advertiser 시 shard 내에서 집계 완료 가능
  → Initiator로 전송되는 중간 데이터 최소화
```

---

## 8. 실시간 과금 쿼리

```sql
-- advertiser별 남은 예산 대비 집행률
SELECT
    advertiser,
    sum(price * sign) / 100 AS spent_today,
    (SELECT daily_budget FROM ads.budgets WHERE advertiser = e.advertiser) AS budget,
    spent_today / budget AS usage_rate
FROM ads.events AS e
WHERE event_type = 'click'
  AND event_time >= today()
GROUP BY advertiser
HAVING usage_rate > 0.8       -- 80% 이상 소진 경고
ORDER BY usage_rate DESC;
```

---

## 9. 성능 최적화 포인트

### 9.1 impression은 샘플링하여 저장

impression이 click보다 100배 많음 → 전량 저장하면 용량 폭증:

```sql
CREATE TABLE ads.impressions_sampled (...)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (...)
SAMPLE BY cityHash64(user_id);    -- 균등 샘플링 기반

INSERT INTO ads.impressions_sampled
SELECT * FROM ads.raw_impressions
WHERE cityHash64(user_id) % 10 = 0;  -- 10% 샘플

-- 쿼리 시 10배 스케일업
SELECT sum(sign) * 10 AS estimated_impressions
FROM ads.impressions_sampled SAMPLE 1;
```

### 9.2 집계 테이블 다단계

```
events (원본, 7일)
  ↓ MV
stats_hourly (1시간 단위, 90일)
  ↓ MV  
stats_daily (1일 단위, 2년)
  ↓ MV
stats_monthly (1달 단위, 영구)
```

오래된 데이터는 정밀도를 낮추고 장기 보관.

---

## 10. 흔한 실수

| 실수 | 문제 | 해결 |
|------|------|------|
| Collapsing 테이블에 `count()` 사용 | +1/-1 행 다 포함되어 2배 | `sum(sign)` 사용 |
| impression 전량 저장 | 디스크 폭증 | 샘플링 |
| Float price 사용 | 누적 시 반올림 오차 | 정수 (센트/원 단위) |
| advertiser를 String | LowCardinality 미적용 → 용량 5배 | LowCardinality |
| ORDER BY 첫 키가 날짜 | advertiser 쿼리 시 전 파티션 스캔 | advertiser를 앞에 |
