# B1. ReplacingMergeTree — FINAL 없이 조회 시 중복 행

## 증상

- `SELECT count()` 결과가 실제 기대값의 2~5배
- 대시보드 매출 합계가 기간에 따라 다른 값이 나옴
- INSERT 직후 조회하면 중복이 없다가, 시간이 지나도 중복이 사라지지 않음

---

## 원인

ReplacingMergeTree는 **머지 시점에만** 중복을 제거한다.

```
INSERT 3회 (user_id=1):
  part_001: (1, 'Bronze', 100, 2024-01-01)
  part_002: (1, 'Silver', 200, 2024-01-10)
  part_003: (1, 'Gold',   500, 2024-01-20)

머지 전: SELECT * → 3개 행 반환  ← 문제!
머지 후: SELECT * → 1개 행 반환 (가장 최신 버전만)
```

```sql
-- 잘못 만든 테이블/쿼리
CREATE TABLE user_profile (
    user_id    UInt64,
    grade      String,
    amount     UInt64,
    updated_at DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;

-- 문제 쿼리 (FINAL 없음)
SELECT user_id, grade, amount
FROM user_profile
WHERE user_id = 1;
-- 결과: 3개 행 반환 (머지 전이라면)
```

머지는 백그라운드에서 비결정적으로 발생한다. 머지가 언제 일어날지 보장할 수 없으므로, FINAL 없이는 항상 중복 가능성이 있다.

---

## 진단

```sql
-- 중복 여부 확인
SELECT user_id, count() AS cnt
FROM user_profile
GROUP BY user_id
HAVING cnt > 1
LIMIT 10;
-- 결과가 나오면 중복 상태

-- FINAL 전후 비교
SELECT count() FROM user_profile;           -- 중복 포함
SELECT count() FROM user_profile FINAL;     -- 중복 제거 후
-- 두 값이 다르면 아직 머지 안 된 중복 존재
```

---

## 해결

### 옵션 1: FINAL 키워드 사용

```sql
-- 조회 시 FINAL 추가 (머지 로직을 쿼리 타임에 적용)
SELECT user_id, grade, amount
FROM user_profile FINAL
WHERE user_id = 1;
```

> **단점**: FINAL은 병렬 처리를 방해한다 (C2 케이스 참고). 데이터가 크면 느림.

### 옵션 2: argMax 패턴 (권장)

```sql
-- FINAL 없이 최신 값만 가져오기
SELECT
    user_id,
    argMax(grade,   updated_at) AS grade,
    argMax(amount,  updated_at) AS amount,
    max(updated_at)             AS updated_at
FROM user_profile
WHERE user_id = 1
GROUP BY user_id;
```

```
원리:
  user_id=1의 모든 행을 읽고 → GROUP BY → argMax가 가장 최신 값 선택
  FINAL처럼 머지 필요 없음, 병렬 처리 유지
```

### 옵션 3: 강제 머지 후 조회 (개발/테스트 환경)

```sql
OPTIMIZE TABLE user_profile FINAL;
-- 이후 FINAL 없이 조회 가능 (새 INSERT 전까지만)
```

### 옵션 4: Materialized View로 argMax 사전 집계

```sql
-- 최신 상태 집계 테이블
CREATE TABLE user_profile_latest (
    user_id    UInt64,
    grade      AggregateFunction(argMax, String, DateTime),
    amount     AggregateFunction(argMax, UInt64, DateTime)
)
ENGINE = AggregatingMergeTree
ORDER BY user_id;

-- MV
CREATE MATERIALIZED VIEW user_profile_mv TO user_profile_latest AS
SELECT
    user_id,
    argMaxState(grade,  updated_at) AS grade,
    argMaxState(amount, updated_at) AS amount
FROM user_profile
GROUP BY user_id;

-- 조회 (FINAL 불필요, 빠름)
SELECT
    user_id,
    argMaxMerge(grade)  AS grade,
    argMaxMerge(amount) AS amount
FROM user_profile_latest
GROUP BY user_id;
```

---

## 예방

```
ReplacingMergeTree 사용 시 조회 패턴을 사전 결정:

  데이터 크기 작음  → FINAL 허용
  데이터 크기 큼    → argMax 패턴 또는 AggregatingMergeTree + MV
  항상 최신 1건만   → argMax(col, version_col) GROUP BY key
```

```sql
-- 팀 내 공통 뷰로 정의해두기
CREATE VIEW user_profile_current AS
SELECT
    user_id,
    argMax(grade,  updated_at) AS grade,
    argMax(amount, updated_at) AS amount
FROM user_profile
GROUP BY user_id;

-- 이후 항상 이 뷰로 조회
SELECT * FROM user_profile_current WHERE user_id = 1;
```
