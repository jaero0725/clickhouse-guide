# A2. 고카디널리티 컬럼 파티셔닝

## 증상

- `system.parts` 파티션 수가 수만 개 이상
- 쿼리 속도 저하 (파티션 메타데이터 로딩만 몇 초)
- `ALTER TABLE DROP PARTITION` 실행 시 수천 개 파티션을 일일이 지정해야 함
- ClickHouse 서버 재시작 시간이 수 분 이상 (파티션 목록 로딩)

---

## 원인

```sql
-- 이렇게 만들면 안 됨
CREATE TABLE user_events (
    event_time DateTime,
    user_id    UInt64,          -- 고유값 수백만
    action     String
)
ENGINE = MergeTree
PARTITION BY user_id            -- ← 치명적 실수
ORDER BY (user_id, event_time);
```

```
user_id 100만 명 → 파티션 100만 개
각 파티션에 파트가 수개씩 → 전체 파트 수백만 개

ClickHouse는 파티션 메타데이터를 메모리에 올림
→ 수백만 파티션 = 수십 GB 메모리 낭비
→ 서버 시작 시 메타데이터 로딩에 수십 분
```

---

## 진단

```sql
-- 파티션 수 확인
SELECT count(DISTINCT partition) AS partition_count
FROM system.parts
WHERE table = 'user_events' AND active;
-- 결과: 1,245,832 (비정상)

-- 메모리에서 파티션 메타데이터가 차지하는 크기 추정
SELECT
    count(DISTINCT partition)                    AS partitions,
    count()                                      AS parts,
    formatReadableSize(sum(bytes_on_disk))        AS total_size
FROM system.parts
WHERE table = 'user_events' AND active;

-- 상위 파티션 샘플
SELECT partition, count() AS parts, sum(rows) AS rows
FROM system.parts
WHERE table = 'user_events' AND active
GROUP BY partition
ORDER BY rows ASC   -- 행이 거의 없는 파티션이 대부분
LIMIT 10;
-- 결과: 파티션당 행이 10~100개에 불과 (비효율)
```

---

## 해결

### 단계 1: 올바른 파티셔닝으로 테이블 재생성

```sql
-- 파티셔닝 목적 재정의:
-- user_id별로 데이터를 관리할 필요가 없다
-- → 시간 기반 TTL/삭제가 목적이라면 월 단위가 맞다

CREATE TABLE user_events_v2 (
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = MergeTree
PARTITION BY toStartOfMonth(event_time)  -- 시간 기반으로 변경
ORDER BY (user_id, event_time);           -- ORDER BY에 user_id 유지
```

### 단계 2: 데이터 이전 (청크 단위)

```sql
-- 전체를 한 번에 이전하면 메모리 부족 위험
-- 월별로 나눠서 이전

INSERT INTO user_events_v2
SELECT * FROM user_events
WHERE event_time >= '2024-01-01' AND event_time < '2024-02-01';

INSERT INTO user_events_v2
SELECT * FROM user_events
WHERE event_time >= '2024-02-01' AND event_time < '2024-03-01';

-- ... 반복
```

### 단계 3: 기존 테이블의 파티션 정리 (임시 방편)

이전 완료 전에 기존 테이블의 파티션을 줄이는 것은 현실적으로 어렵다. 가능한 빨리 마이그레이션하는 것이 최선.

```sql
-- 오래된 파티션부터 DROP (사용자 ID 기반이라 날짜 필터 불가)
-- 임시: 특정 user_id 범위를 직접 DROP
ALTER TABLE user_events DROP PARTITION '12345';
```

---

## 예방

```sql
-- 파티셔닝 컬럼 고유값 수 사전 확인
SELECT uniqExact(user_id) AS unique_users FROM source_table;
-- 1,000 이하: 파티셔닝 허용 (신중히)
-- 1,000 이상: 절대 파티셔닝 금지

-- 올바른 파티셔닝 후보:
PARTITION BY toStartOfMonth(event_time)  -- ✅ 고유값 수십 개
PARTITION BY toYYYYMM(event_time)        -- ✅ 동일
PARTITION BY status                      -- ⚠️ 상태값이 10개 이하라면 허용
PARTITION BY user_id                     -- ❌ 수백만 고유값
PARTITION BY toDate(event_time)          -- ❌ A1 참고
```

**파티셔닝 기준: 고유값 1,000개 이하, 데이터 관리(TTL/삭제) 목적에 한해서만**
