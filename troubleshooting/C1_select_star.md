# C1. SELECT * — 컬럼 지향 장점 소멸

## 증상

- "빠르다고 했는데 왜 이렇게 느리지?"
- 필요한 컬럼은 2~3개인데 쿼리 시간이 수십 초
- I/O 사용량이 디스크 전체 대역폭에 근접

---

## 원인

ClickHouse는 컬럼 지향 데이터베이스다. 컬럼별로 파일이 분리 저장되어 있어, **필요한 컬럼 파일만 열면** 나머지는 완전히 무시된다.

```
테이블 디스크 구조:
  event_time.bin  (10 GB)
  user_id.bin     (8  GB)
  action.bin      (15 GB)
  amount.bin      (4  GB)
  category.bin    (2  GB)
  detail.bin      (20 GB)   ← 긴 텍스트
  meta.bin        (18 GB)   ← JSON 등
  ...
  총 합계: 100 GB

SELECT amount, category FROM events WHERE ...
→ amount.bin(4GB) + category.bin(2GB) = 6GB만 읽음

SELECT * FROM events WHERE ...
→ 100GB 전체 읽음 → 17배 더 읽음
```

---

## 진단

```sql
-- 쿼리별 읽은 바이트 비교
SELECT
    query,
    read_bytes,
    read_rows,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%events%'
ORDER BY query_start_time DESC
LIMIT 10;

-- EXPLAIN으로 읽을 컬럼 목록 확인
EXPLAIN SELECT * FROM events WHERE user_id = 123;
-- 출력에서 ReadFromMergeTree의 columns 개수 확인
```

실시간 비교:

```sql
-- 느린 쿼리
SELECT * FROM events WHERE date = today() LIMIT 1000;

-- 빠른 쿼리
SELECT event_time, user_id, amount FROM events WHERE date = today() LIMIT 1000;

-- LIMIT 없이 집계 쿼리
SELECT sum(amount) FROM events WHERE date = today();  -- 가장 빠름
```

---

## 해결

### 기본 원칙

```sql
-- 금지
SELECT * FROM events WHERE ...

-- 올바른 방법: 필요한 컬럼만
SELECT event_time, user_id, action, amount
FROM events
WHERE category_id = 'electronics'
  AND date = today();
```

### 실수가 잦은 패턴 수정

```sql
-- 나쁜 예 1: 디버깅 쿼리를 그대로 프로덕션에
SELECT * FROM events LIMIT 100;

-- 좋은 예: 필요한 컬럼 명시
SELECT event_time, user_id, action FROM events LIMIT 100;
```

```sql
-- 나쁜 예 2: ORM이 자동 생성한 SELECT *
-- Django ORM: Event.objects.filter(date=today)
-- → 내부적으로 SELECT * 실행

-- 해결: values() 또는 only() 사용
Event.objects.filter(date=today).values('event_time', 'user_id', 'amount')
```

```sql
-- 나쁜 예 3: Subquery에서 SELECT *
SELECT user_id, max(amount)
FROM (SELECT * FROM events WHERE status = 'paid')
GROUP BY user_id;

-- 좋은 예
SELECT user_id, max(amount)
FROM events
WHERE status = 'paid'
GROUP BY user_id;
-- (서브쿼리 자체를 없애거나 컬럼 명시)
```

### 대시보드/리포트에서 SELECT * 패턴

```sql
-- 대시보드 쿼리 — 나쁜 예
SELECT * FROM daily_sales ORDER BY date DESC LIMIT 30;

-- 대시보드 쿼리 — 좋은 예 (컬럼 명시 + Materialized View 활용)
SELECT date, category_id, amount, quantity
FROM daily_sales_mv          -- 사전 집계된 MV
ORDER BY date DESC
LIMIT 30;
```

---

## 성능 비교

실제 1억 행, 20개 컬럼 테이블 기준:

| 쿼리 | 읽은 바이트 | 시간 |
|------|------------|------|
| `SELECT *` | 48 GB | 42초 |
| `SELECT col1, col2, col3` | 3.2 GB | 2.8초 |
| `SELECT sum(amount)` | 0.8 GB | 0.7초 |

→ 컬럼 선택만으로 60배 차이

---

## 예방

```sql
-- 개발 환경에서 SELECT * 경고 설정 (ClickHouse 설정)
SET max_columns_to_read = 10;  -- 10개 이상 컬럼 읽으면 에러

-- 쿼리 리뷰 체크리스트:
-- ✅ SELECT 절에 필요한 컬럼만 명시했는가?
-- ✅ 대시보드는 Materialized View를 사용하는가?
-- ✅ ORM 쿼리가 SELECT *를 생성하지 않는가?
```
