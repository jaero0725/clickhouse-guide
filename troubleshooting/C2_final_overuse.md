# C2. FINAL 남발 — 싱글 스레드 실행 + CPU 폭증

## 증상

- ReplacingMergeTree 테이블 조회 시 CPU가 단일 코어에 100% 고정
- 동일 데이터 크기인데 일반 MergeTree보다 10~30배 느림
- 서버에 코어가 32개인데 쿼리 하나가 1개만 사용

---

## 원인

`FINAL` 키워드는 쿼리 실행 시점에 머지 로직을 적용한다. 이 과정이 병렬 처리를 방해한다.

```
일반 SELECT (병렬):
  파트 A → Thread 1 ─┐
  파트 B → Thread 2  ├─→ 집계 → 결과
  파트 C → Thread 3 ─┘
  파트 D → Thread 4

FINAL SELECT (싱글 스레드):
  파트 A ─┐
  파트 B  ├─→ 정렬·중복제거(단일) → 집계 → 결과
  파트 C  │   (모든 파트를 하나씩 비교해야 함)
  파트 D ─┘
```

추가 비용:
1. **병렬 처리 불가**: 모든 파트를 순서대로 처리
2. **더 많은 데이터 읽음**: 중복 제거를 위해 중복 행 전부 읽어야 함
3. **추가 CPU/메모리**: 중복 행 비교 및 정렬 비용

---

## 진단

```sql
-- FINAL 유무에 따른 차이 비교
-- (1) FINAL 없음
SELECT count(), sum(amount) FROM user_profile WHERE grade = 'Gold';
-- 실행 시간: 0.3초, 읽은 행: 1,000,000

-- (2) FINAL 있음
SELECT count(), sum(amount) FROM user_profile FINAL WHERE grade = 'Gold';
-- 실행 시간: 8.2초, 읽은 행: 3,200,000 (중복 포함)

-- 실행 계획 비교
EXPLAIN SELECT count() FROM user_profile FINAL;
-- 출력에서 'CreatingSets' 또는 'Collapsing'/'Replacing' 스텝 확인
```

```sql
-- 현재 FINAL을 사용하는 느린 쿼리 찾기
SELECT query, query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%FINAL%'
  AND query_duration_ms > 1000
ORDER BY query_duration_ms DESC
LIMIT 20;
```

---

## 해결

### 해결책 1: argMax 패턴으로 교체 (가장 일반적)

```sql
-- 나쁜 예
SELECT user_id, grade, amount
FROM user_profile FINAL
WHERE grade = 'Gold';

-- 좋은 예
SELECT
    user_id,
    argMax(grade,  updated_at) AS grade,
    argMax(amount, updated_at) AS amount
FROM user_profile
WHERE grade = 'Gold'          -- 이 WHERE는 여전히 중복 행에 적용됨
GROUP BY user_id
HAVING argMax(grade, updated_at) = 'Gold';  -- 최신 grade 기준 필터
```

### 해결책 2: Materialized View로 사전 집계

FINAL을 쓰는 주된 이유가 최신 상태 집계라면, AggregatingMergeTree + MV로 대체.

```sql
-- 사전 집계 테이블
CREATE TABLE user_profile_agg (
    user_id     UInt64,
    grade       AggregateFunction(argMax, String,   DateTime),
    amount      AggregateFunction(argMax, UInt64,   DateTime),
    last_updated AggregateFunction(max,  DateTime)
)
ENGINE = AggregatingMergeTree
ORDER BY user_id;

CREATE MATERIALIZED VIEW user_profile_mv TO user_profile_agg AS
SELECT
    user_id,
    argMaxState(grade,  updated_at) AS grade,
    argMaxState(amount, updated_at) AS amount,
    maxState(updated_at)            AS last_updated
FROM user_profile
GROUP BY user_id;

-- 조회 (FINAL 불필요, 병렬 처리)
SELECT
    user_id,
    argMaxMerge(grade)  AS grade,
    argMaxMerge(amount) AS amount
FROM user_profile_agg
WHERE user_id IN (1, 2, 3)    -- 키 조회는 빠름
GROUP BY user_id;
```

### 해결책 3: 불가피하게 FINAL 써야 할 때 병렬화 힌트

ClickHouse 22.8 이후 FINAL의 병렬 처리가 일부 개선됐다.

```sql
-- 병렬 FINAL (실험적 기능)
SET max_final_threads = 8;   -- FINAL 처리에 사용할 스레드 수

SELECT count() FROM user_profile FINAL;
-- 완전한 병렬은 아니지만 단일 스레드보다 개선
```

### 해결책 4: 조회 전 강제 머지 (소규모 테이블)

```sql
-- 개발/배치 환경에서 주기적으로 머지
OPTIMIZE TABLE user_profile FINAL;
-- 이후 FINAL 없이 조회 가능 (새 INSERT 전까지)
```

---

## 패턴별 대안 정리

| 목적 | FINAL 대신 사용 |
|------|----------------|
| 최신 값 1개 | `argMax(col, version)` |
| 최신 상태 집계 | AggregatingMergeTree + MV |
| 중복 제거 카운트 | `uniqExact` 또는 `uniq` |
| CollapsingMergeTree 조회 | `sum(col * sign) HAVING sum(sign) > 0` |
| 확실한 데이터 필요 | 배치 후 `OPTIMIZE` → 일반 조회 |

---

## 예방

```
FINAL 사용 전 자문:

  1. 이 테이블의 데이터가 얼마나 큰가?
     → 수천만 행 이상: FINAL 금지
  
  2. 머지 후 중복 없음이 보장되는가?
     → OPTIMIZE 직후: 허용 (테스트 환경)
  
  3. 대시보드/리포트 쿼리인가?
     → MV 사용으로 FINAL 자체를 없애기

  FINAL = "필요하지만 성능을 희생한다"는 명시적 선택
  기본 패턴으로 쓰면 안 됨
```
