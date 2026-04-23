# B3. SummingMergeTree — 머지 후 String 컬럼 값 소실

## 증상

- 머지 전에는 정상적으로 보이던 카테고리명, 상태값 등이 머지 후 임의 값으로 변경
- `category_name`이 'electronics'였는데 머지 후 'fashion'으로 바뀜
- 집계 숫자는 맞는데 함께 조회한 문자열 컬럼이 예측 불가

---

## 원인

SummingMergeTree는 머지 시 **숫자 타입** 컬럼은 합산하고, **나머지 컬럼(ORDER BY 키 제외)은 첫 번째 행의 값을 사용**한다.

```sql
CREATE TABLE sales (
    date          Date,
    category_id   UInt32,
    category_name String,      -- ← 비집계 컬럼
    amount        UInt64,
    quantity      UInt32
)
ENGINE = SummingMergeTree
ORDER BY (category_id, date);
```

```
INSERT 2회:
  (2024-01, 1, 'electronics', 10000, 5)
  (2024-01, 1, 'Electronics', 20000, 8)  ← category_name 대소문자 다름

머지 후:
  (2024-01, 1, 'electronics', 30000, 13)
  → amount, quantity 합산 OK
  → category_name: 첫 번째 행의 값 사용 ('electronics')
  → 'Electronics'는 소실됨
```

더 위험한 경우:

```
(2024-01, 1, 'electronics', 10000, 5)
(2024-01, 1, 'WRONG_VALUE', 20000, 8)   ← 잘못된 데이터가 먼저 들어옴

머지 후: category_name = 'WRONG_VALUE' (30000, 13과 함께)
→ 수정 불가 (어떤 행이 "첫 번째"인지 보장 없음)
```

---

## 진단

```sql
-- category_id별 category_name 종류 확인 (1개 이상이면 의심)
SELECT category_id, groupArray(DISTINCT category_name) AS names
FROM sales
GROUP BY category_id
HAVING length(names) > 1;

-- 머지 전후 비교 (FINAL로 머지 효과 시뮬레이션)
SELECT category_id, category_name, sum(amount)
FROM sales
GROUP BY category_id, category_name;
-- vs
SELECT category_id, category_name, sum(amount)
FROM sales FINAL
GROUP BY category_id, category_name;
```

---

## 해결

### 해결책 1: 비집계 컬럼은 ORDER BY에 포함시키기

비집계 컬럼이 식별자 역할을 한다면 ORDER BY 키에 포함시킨다.

```sql
CREATE TABLE sales_v2 (
    date          Date,
    category_id   UInt32,
    category_name LowCardinality(String),  -- ORDER BY에 포함
    amount        UInt64,
    quantity      UInt32
)
ENGINE = SummingMergeTree
ORDER BY (category_id, category_name, date);
-- category_name이 다르면 다른 행으로 취급 → 합산 안 됨
```

> 단, category_name이 카디널리티가 높다면 인덱스 효율이 떨어짐.

### 해결책 2: 집계할 컬럼만 명시 (권장)

SummingMergeTree는 집계할 컬럼 목록을 명시할 수 있다. 명시된 컬럼만 합산하고 나머지는 ORDER BY 키처럼 취급한다.

```sql
CREATE TABLE sales_v3 (
    date          Date,
    category_id   UInt32,
    category_name LowCardinality(String),
    amount        UInt64,
    quantity      UInt32
)
ENGINE = SummingMergeTree(amount, quantity)  -- 이 컬럼만 합산
ORDER BY (category_id, date);
-- category_name: 첫 번째 행 값 사용 (여전히 비결정적)
```

**진짜 해결: category_name을 별도 차원 테이블로 분리**

```sql
-- 차원 테이블 (카테고리 메타데이터)
CREATE TABLE category_dim (
    category_id   UInt32,
    category_name String,
    updated_at    DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY category_id;

-- 팩트 테이블 (숫자만)
CREATE TABLE sales_fact (
    date        Date,
    category_id UInt32,
    amount      UInt64,
    quantity    UInt32
)
ENGINE = SummingMergeTree
ORDER BY (category_id, date);

-- 조회 시 JOIN
SELECT s.date, c.category_name, s.amount, s.quantity
FROM sales_fact s
JOIN (
    SELECT category_id, argMax(category_name, updated_at) AS category_name
    FROM category_dim GROUP BY category_id
) c ON s.category_id = c.category_id;
```

### 해결책 3: AggregatingMergeTree로 교체 (유연성 필요 시)

문자열을 포함한 복잡한 집계가 필요하다면 SummingMergeTree보다 AggregatingMergeTree가 적합하다.

```sql
CREATE TABLE sales_agg (
    date        Date,
    category_id UInt32,
    amount      AggregateFunction(sum, UInt64),
    quantity    AggregateFunction(sum, UInt32)
)
ENGINE = AggregatingMergeTree
ORDER BY (category_id, date);

-- MV
CREATE MATERIALIZED VIEW sales_agg_mv TO sales_agg AS
SELECT
    toDate(event_time)  AS date,
    category_id,
    sumState(amount)    AS amount,
    sumState(quantity)  AS quantity
FROM raw_events
GROUP BY date, category_id;

-- 조회
SELECT date, category_id, sumMerge(amount), sumMerge(quantity)
FROM sales_agg
GROUP BY date, category_id;
```

---

## 예방

```
SummingMergeTree 설계 원칙:

  ✅ ORDER BY 키: 집계 차원 (집계 기준이 되는 컬럼들)
  ✅ 집계 컬럼: 숫자 타입 (UInt, Int, Float)만
  ✅ 문자열 메타데이터: 별도 차원 테이블로 분리
  ❌ String/LowCardinality를 합산 대상 컬럼 옆에 두지 않기

  SummingMergeTree(col1, col2) 형태로 집계 컬럼 명시 권장
```
