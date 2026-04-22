# 예제 2. Observability — 애플리케이션 로그와 메트릭

수백 개 서비스가 초당 수만 건의 로그를 쏟아내는 Observability 시나리오. ClickHouse가 가장 잘하는 영역이다.

---

## 1. 요구사항

- 초당 50,000건 로그 수집
- 각 로그: timestamp, level, service, trace_id, message, attributes(JSON)
- 쿼리 패턴:
  - 특정 서비스의 최근 ERROR 로그 조회
  - trace_id로 분산 추적
  - 시간대별 에러율 대시보드
- 14일 후 데이터 자동 삭제

---

## 2. 로그 테이블

```sql
CREATE TABLE logs.app_logs (
    timestamp    DateTime64(3),            -- 밀리초 정밀도
    service      LowCardinality(String),   -- 약 수백 개 서비스
    level        LowCardinality(String),   -- DEBUG, INFO, WARN, ERROR
    trace_id     String,                   -- OpenTelemetry trace ID
    span_id      String,
    host         LowCardinality(String),
    message      String CODEC(ZSTD(3)),    -- 긴 텍스트, 고압축
    attributes   Map(String, String)       -- 동적 키-값
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)             -- 일 단위 (14개만 유지)
ORDER BY (service, level, timestamp)
TTL toDateTime(timestamp) + INTERVAL 14 DAY DELETE
SETTINGS index_granularity = 8192;

-- trace_id로 빠른 검색을 위한 bloom filter
ALTER TABLE logs.app_logs
ADD INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 1;
```

### 설계 근거

```
ORDER BY (service, level, timestamp)
         │         │         │
         │         │         └── 시간 범위 필터용
         │         └── ERROR 로그만 필터할 때 대량 skip
         └── 특정 서비스 쿼리 시 첫 번째 레벨에서 skip

PARTITION BY toDate(timestamp)
→ 14일치만 유지 → TTL 자동 정리 + 파티션 DROP 효율
→ 파티션 수 14개로 제한 → 과도한 파티셔닝 아님
```

---

## 3. Async Insert로 대량 수집

Log agent(Vector, Fluent Bit 등)가 수백 개 인스턴스에서 직접 INSERT:

```
Agent 1  ─┐
Agent 2  ─┤→  ClickHouse (async_insert=1)
Agent 3  ─┤                │
...      ─┤                ▼
Agent 500─┘          서버 버퍼링 (1초 or 10MB)
                           │
                           ▼
                      파트 1개 생성
```

```sql
-- 에이전트용 유저 설정
CREATE USER log_agent IDENTIFIED BY '...';
ALTER USER log_agent SETTINGS
    async_insert = 1,
    wait_for_async_insert = 1,        -- 에러 인지 필요
    async_insert_max_data_size = 10000000,  -- 10MB
    async_insert_busy_timeout_ms = 1000;    -- 1초
```

### Async Insert가 아니면 안 되는 이유

```
건 단위 INSERT 50,000 qps
  → 초당 50,000개 파트 생성
  → 1분 내 3,000개 돌파 → "Too many parts"
  → 데이터 수집 중단

Async Insert:
  → 서버가 여러 INSERT를 1초간 버퍼링
  → 1초마다 파트 1개 (초당 1개)
  → 안정적
```

---

## 4. 주요 쿼리

### 쿼리 1: 특정 서비스의 ERROR 로그

```sql
SELECT timestamp, message, attributes
FROM logs.app_logs
WHERE service = 'payment-api'
  AND level = 'ERROR'
  AND timestamp >= now() - INTERVAL 1 HOUR
ORDER BY timestamp DESC
LIMIT 100;
```

**ORDER BY 키 (service, level, timestamp) 효과:**

```
파트 내부:
 ┌ service=auth-api,    level=ERROR, ...
 │  (많은 granule)
 ├ service=payment-api, level=DEBUG, ...
 │  ← 여기부터 skip
 ├ service=payment-api, level=ERROR, ...
 │  ← 요청 범위! 여기만 읽음
 ├ service=payment-api, level=INFO,  ...
 │  ← 여기부터 skip
 └ service=user-api,    level=ERROR, ...

인덱스 이진 탐색으로 해당 granule 범위 바로 점프
```

### 쿼리 2: Trace ID로 분산 추적

```sql
SELECT timestamp, service, level, message
FROM logs.app_logs
WHERE trace_id = 'abc-123-def-456'
ORDER BY timestamp ASC;
```

`trace_id`는 ORDER BY 키 밖에 있음 → 기본적으로 풀 스캔이지만, **bloom filter**로 대부분 granule을 skip:

```
파트 1: bloom_filter → "이 trace_id 없을 것" → skip
파트 2: bloom_filter → "여기 있을 수 있음" → 읽음
파트 3: bloom_filter → "없음" → skip
...
99% skip 효과
```

### 쿼리 3: 시간대별 에러율 (사전 집계)

```sql
CREATE TABLE logs.error_rate_5m (
    time_5m     DateTime,
    service     LowCardinality(String),
    total       UInt64,
    errors      UInt64
)
ENGINE = SummingMergeTree
ORDER BY (service, time_5m);

CREATE MATERIALIZED VIEW logs.error_rate_mv
TO logs.error_rate_5m AS
SELECT
    toStartOfInterval(timestamp, INTERVAL 5 MINUTE) AS time_5m,
    service,
    count() AS total,
    countIf(level = 'ERROR') AS errors
FROM logs.app_logs
GROUP BY time_5m, service;
```

대시보드 쿼리는 원본 대신 집계 테이블:

```sql
SELECT time_5m, service, errors / total AS error_rate
FROM logs.error_rate_5m
WHERE time_5m >= now() - INTERVAL 1 HOUR
ORDER BY time_5m, service;
```

원본 스캔보다 수백 배 빠름.

---

## 5. Map 타입 활용

```sql
-- attributes는 Map(String, String)
SELECT attributes['user_id'], attributes['http_status']
FROM logs.app_logs
WHERE service = 'payment-api'
  AND attributes['http_status'] = '500'
  AND timestamp >= now() - INTERVAL 1 HOUR;
```

Map 필터링은 풀 스캔이므로, 자주 쓰는 attribute는 **별도 컬럼으로 승격**이 권장:

```sql
ALTER TABLE logs.app_logs
ADD COLUMN http_status LowCardinality(String) 
DEFAULT attributes['http_status'];
```

---

## 6. 운영 체크리스트

```sql
-- 오늘 수집량
SELECT count() FROM logs.app_logs 
WHERE timestamp >= today();

-- 서비스별 로그량 Top 10
SELECT service, count() AS logs
FROM logs.app_logs
WHERE timestamp >= today()
GROUP BY service ORDER BY logs DESC LIMIT 10;

-- Async Insert 현황
SELECT * FROM system.asynchronous_insert_log
WHERE event_time >= now() - INTERVAL 10 MINUTE
ORDER BY event_time DESC LIMIT 20;

-- 파트 수 (문제 감지)
SELECT count() AS parts, min(level), max(level)
FROM system.parts
WHERE database = 'logs' AND table = 'app_logs' AND active;
```

---

## 7. 흔한 실수와 해결

| 실수 | 증상 | 해결 |
|------|------|------|
| 동기 INSERT 건단위 | Too many parts | Async Insert 활성화 |
| `PARTITION BY toStartOfHour` | 파티션 수백 개, 머지 분산 | Day 단위로 변경 |
| `message String` 인덱스 시도 | 인덱스 크기 폭증 | Full text search는 Tantivy/OpenSearch |
| Map 자주 필터링 | 쿼리 느림 | 별도 컬럼 승격 |
| TTL 안 걸음 | 디스크 계속 증가 | `TTL ... DELETE` 추가 |
