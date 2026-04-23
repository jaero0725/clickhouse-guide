# C4. DISTINCT vs uniq — DAU 쿼리 메모리 폭발

## 증상

- DAU(일별 활성 사용자) 집계 쿼리가 10초 이상 소요
- 메모리 사용량이 수 GB
- 분산 환경에서는 OOM으로 쿼리 실패
- 사용자가 수백만~수천만 명인 서비스

---

## 원인

`COUNT(DISTINCT user_id)`는 **정확한 고유 카운트**를 위해 모든 user_id를 메모리에 올려 해시셋을 만든다.

```
1억 이벤트, 500만 user_id:
  COUNT(DISTINCT user_id)
  → 500만 user_id × 8바이트 = 40MB (최소)
  → 해시맵 오버헤드 포함 시 ~160~400MB
  → 분산 환경: 샤드 수 × 위 크기 = 수 GB
```

반면 `uniq(user_id)`는 **HyperLogLog** 알고리즘을 사용한다.

```
HyperLogLog 원리:
  - 모든 값을 저장하지 않음
  - 해시값의 선행 0 비트 수만 추적
  - 고정 크기 스케치(2.5KB)만 사용
  - 오차: ~2% (기본), 조정 가능
```

---

## 진단

```sql
-- 느린 DAU 쿼리
SELECT
    toDate(event_time) AS date,
    COUNT(DISTINCT user_id) AS dau       -- ← 문제
FROM events
WHERE event_time >= today() - 30
GROUP BY date
ORDER BY date;

-- query_log에서 메모리 사용량 확인
SELECT
    query,
    query_duration_ms,
    formatReadableSize(memory_usage) AS mem
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%DISTINCT user_id%'
ORDER BY query_start_time DESC
LIMIT 5;
```

---

## 해결

### 해결책 1: uniq 함수로 교체 (즉시 적용)

```sql
-- 나쁜 예
SELECT
    toDate(event_time) AS date,
    COUNT(DISTINCT user_id) AS dau
FROM events
WHERE event_time >= today() - 30
GROUP BY date;

-- 좋은 예 (~2% 오차, 수백 배 빠름)
SELECT
    toDate(event_time) AS date,
    uniq(user_id) AS dau                   -- HyperLogLog
FROM events
WHERE event_time >= today() - 30
GROUP BY date;
```

### uniq 함수 종류

```sql
uniq(col)           -- 기본 HyperLogLog (~2% 오차, 2.5KB)
uniqHLL12(col)      -- HyperLogLog12 (~1.6% 오차, 2.5KB)
uniqExact(col)      -- 정확한 카운트 (DISTINCT와 동일, 고메모리)
uniqCombined(col)   -- 적은 카디널리티에서 정확, 많을 때 HLL
uniqCombined64(col) -- uniqCombined의 64비트 버전 (더 정확)
```

선택 기준:

```
카디널리티 < 100,000      → uniqCombined (정확도 높음, 빠름)
카디널리티 100만 이상      → uniq 또는 uniqHLL12
정확도 1% 이내 필요        → uniqCombined64
완전 정확한 값 필수        → uniqExact (메모리 주의)
```

### 해결책 2: AggregatingMergeTree + uniqState MV

반복되는 DAU 쿼리라면 사전 집계로 응답 시간을 ms 단위로 낮춘다.

```sql
-- 사전 집계 테이블
CREATE TABLE dau_daily (
    date    Date,
    dau     AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY date;

-- Materialized View (INSERT될 때마다 자동 집계)
CREATE MATERIALIZED VIEW dau_daily_mv TO dau_daily AS
SELECT
    toDate(event_time)  AS date,
    uniqState(user_id)  AS dau
FROM events
GROUP BY date;

-- 조회 (빠름: 30일치 = 30행만 읽음)
SELECT
    date,
    uniqMerge(dau) AS dau
FROM dau_daily
WHERE date >= today() - 30
GROUP BY date
ORDER BY date;
```

```
성능 비교 (500만 유저, 30일):
  COUNT(DISTINCT) 직접 집계: 12초, 1.2GB
  uniq() 직접 집계:          0.8초, 12MB
  uniqMerge() on MV:        0.02초, 1MB (600배 차이)
```

### 해결책 3: 정밀도 필요 구간만 uniqExact

```sql
-- 대부분의 경우 uniq로 처리하되,
-- 보고서에서 정확한 값이 필요한 특정 날짜만 uniqExact
SELECT uniqExact(user_id) FROM events WHERE date = '2024-01-15';
-- 하루치만 → 메모리 부담 적음
```

---

## 성능 비교

| 함수 | 오차율 | 메모리 | 속도 |
|------|--------|--------|------|
| `COUNT(DISTINCT)` | 0% | 수백 MB~GB | 느림 |
| `uniqExact` | 0% | 수백 MB~GB | 느림 (동일) |
| `uniqCombined` | ~0.2% | 수 MB | 빠름 |
| `uniq` | ~2% | 2.5 KB | 매우 빠름 |
| `uniqHLL12` | ~1.6% | 2.5 KB | 매우 빠름 |
| `uniqMerge` (MV) | ~2% | 수 KB | 초고속 |

---

## 예방

```sql
-- 팀 내 코드 리뷰 기준:
-- COUNT(DISTINCT col) → "카디널리티 얼마인가?" 질문
-- 10만 이하: 허용
-- 10만 이상: uniq/uniqCombined 교체 요청

-- ClickHouse 사용 정책으로 등록:
-- "DAU/MAU 집계는 항상 uniqState MV 경유"
```
