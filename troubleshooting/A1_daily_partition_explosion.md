# A1. 일 단위 파티셔닝 → 파트 폭증

## 증상

```
DB::Exception: Too many parts (3002). Merges are processing significantly slower than inserts.
```

- INSERT 속도가 점점 느려지다가 결국 에러로 중단
- `system.parts`를 조회하면 파트 수가 수천~수만 개
- 백그라운드 머지가 따라가지 못하는 상태

---

## 원인

```sql
-- 이렇게 테이블을 만들었을 때
CREATE TABLE events (
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = MergeTree
PARTITION BY toDate(event_time)   -- ← 문제
ORDER BY (user_id, event_time);
```

**파티션 경계를 넘어서는 머지는 불가능하다.**

```
날짜별 파티션:
  2024-01-01 │ part_1(1만행) part_2(9천행) part_3(8천행) ... (머지 가능)
  2024-01-02 │ part_1(1만행) part_2(9천행) ...               (머지 가능)
  2024-01-03 │ ...
  ...
  2024-12-31 │ ...

1년 운영 시: 365일 × 파티션당 파트 수 = 수천~수만 파트
```

하루에 INSERT가 여러 번 일어나면 파티션 `2024-01-01` 안에서만 머지가 일어나야 한다. 파티션 수가 늘어날수록 머지 대상이 분산되어 ClickHouse의 백그라운드 머지가 따라가지 못한다.

---

## 진단

```sql
-- 파티션별 파트 수 확인
SELECT
    partition,
    count()          AS part_count,
    sum(rows)        AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS size_on_disk
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
ORDER BY part_count DESC
LIMIT 20;

-- 결과 예시 (문제 상황):
-- partition   │ part_count │ total_rows │ size_on_disk
-- 2024-01-01  │     312    │  3,200,000 │ 245 MB
-- 2024-01-02  │     298    │  2,900,000 │ 231 MB
-- 2024-01-03  │     341    │  3,100,000 │ 248 MB
-- ...
-- → 파티션당 파트 300개 이상: 정상은 10개 이하
```

```sql
-- 전체 파트 수
SELECT count() FROM system.parts
WHERE table = 'events' AND active;
-- 결과: 11,000 (정상은 수십~수백)
```

---

## 해결

### 단계 1: 즉각 완화 — OPTIMIZE 강제 머지

```sql
-- 특정 파티션 강제 머지
OPTIMIZE TABLE events PARTITION '2024-01-01' FINAL;

-- 전체 강제 머지 (테이블이 크면 매우 오래 걸림, 주의)
OPTIMIZE TABLE events FINAL;
```

> **주의**: 대용량 테이블에서 `OPTIMIZE ... FINAL`은 몇 시간이 걸릴 수 있다. 프로덕션에서는 파티션 단위로 나눠서 실행.

### 단계 2: 근본 해결 — 파티셔닝 변경

파티셔닝 키는 테이블 생성 후 직접 변경 불가. 새 테이블 생성 + 데이터 마이그레이션 필요.

```sql
-- 1. 새 테이블 생성 (월 단위 파티셔닝)
CREATE TABLE events_v2 (
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = MergeTree
PARTITION BY toStartOfMonth(event_time)   -- 월 단위로 변경
ORDER BY (user_id, event_time);

-- 2. 데이터 마이그레이션 (INSERT SELECT)
INSERT INTO events_v2
SELECT * FROM events;

-- 3. 트래픽 전환 후 기존 테이블 삭제
RENAME TABLE events TO events_old, events_v2 TO events;
DROP TABLE events_old;
```

### 단계 3: 파트 수 모니터링

```sql
-- 정상 기준: 파티션당 파트 수 < 10
SELECT
    partition,
    count() AS parts
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
HAVING parts > 10
ORDER BY parts DESC;

-- 아무 결과도 안 나오면 정상
```

---

## 예방

```sql
-- 올바른 파티셔닝 기준
PARTITION BY toStartOfMonth(event_time)   -- 보통 이것으로 충분
PARTITION BY toYYYYMM(event_time)         -- 동일한 효과

-- 파티션 수 계산:
-- 3년 데이터 보관 → 36개 파티션 (적절)
-- 3년 데이터 + 일 단위 → 1,095개 파티션 (위험)
```

**파티셔닝 목적은 쿼리 가속이 아니라 데이터 관리(TTL, DROP)이다.**

```
파티션 수 권장 기준:
  < 100개   : 안전
  100~500개 : 주의
  500개 이상: 위험, 재설계 검토
```
