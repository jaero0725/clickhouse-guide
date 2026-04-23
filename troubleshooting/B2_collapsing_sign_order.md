# B2. CollapsingMergeTree — sign 순서 꼬임

## 증상

- 재고/잔액/카운터 집계가 음수로 나옴
- `sum(amount * sign)` 결과가 0이 되어야 하는데 다른 값
- 취소 처리 후 오히려 값이 증가

---

## 원인

CollapsingMergeTree는 **같은 정렬 키(ORDER BY)에 대해 sign=1인 행과 sign=-1인 행을 쌍으로 만들어 상쇄**시킨다. 단, 머지 시 정렬 키가 동일한 행들 사이에서만 상쇄가 일어난다.

### 상쇄 규칙

```
sign=1  (삽입)   → 현재 상태 기록
sign=-1 (취소)   → 기존 상태 취소

상쇄 조건:
  1. 두 행의 ORDER BY 키가 완전히 동일
  2. 한 행은 sign=1, 다른 행은 sign=-1
  3. 같은 파트 내에 존재 (머지가 일어나야 상쇄)
```

### 흔한 실수 — 취소 행의 키가 다름

```sql
CREATE TABLE inventory (
    item_id     UInt64,
    warehouse   String,
    quantity    Int32,
    updated_at  DateTime,   -- ← ORDER BY에 포함
    sign        Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (item_id, warehouse, updated_at);  -- updated_at 포함이 문제

-- 원래 상태 INSERT
INSERT INTO inventory VALUES (1, 'Seoul', 100, '2024-01-01 10:00:00', 1);

-- 상태 변경 시 취소 행
INSERT INTO inventory VALUES (1, 'Seoul', 100, '2024-01-01 11:00:00', -1);
--                                                ^^^^^^^^^^^^^^^^^^^
-- updated_at이 다름 → ORDER BY 키가 다름 → 상쇄 불가!
-- 새 상태
INSERT INTO inventory VALUES (1, 'Seoul', 80,  '2024-01-01 11:00:00', 1);
```

```
머지 후 결과:
  (1, Seoul, 100, 2024-01-01 10:00:00, sign=1)  ← 취소 안 됨 (키 불일치)
  (1, Seoul, 100, 2024-01-01 11:00:00, sign=-1) ← 원본 없음
  (1, Seoul, 80,  2024-01-01 11:00:00, sign=1)

sum(quantity * sign) = 100 + (-100) + 80 = 80  → 운 좋게 맞음
하지만 다음 취소 시 또 꼬임
```

---

## 진단

```sql
-- sign별 행 수 확인 (1:-1 = 1:1이어야 정상, 마지막 상태 1개 초과분 허용)
SELECT sign, count() AS cnt, sum(quantity) AS total_qty
FROM inventory
GROUP BY sign;
-- sign=1:  count=150, total=12,500
-- sign=-1: count=147, total=12,200
-- 취소 안 된 3건 존재

-- 짝이 없는 취소(-1) 행 찾기
SELECT item_id, warehouse, updated_at, quantity, sign
FROM inventory
WHERE sign = -1
AND (item_id, warehouse, updated_at) NOT IN (
    SELECT item_id, warehouse, updated_at FROM inventory WHERE sign = 1
)
LIMIT 10;
```

---

## 해결

### 해결책 1: ORDER BY에서 가변 컬럼(updated_at) 제거

```sql
-- 올바른 설계: ORDER BY에 변하는 값 넣지 않기
CREATE TABLE inventory_v2 (
    item_id     UInt64,
    warehouse   LowCardinality(String),
    quantity    Int32,
    updated_at  DateTime,
    sign        Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (item_id, warehouse);   -- 식별자만 포함
```

```sql
-- 올바른 상태 변경 패턴
-- 1. 원래 상태와 동일한 ORDER BY 키로 sign=-1 INSERT
INSERT INTO inventory_v2 VALUES (1, 'Seoul', 100, '2024-01-01 10:00:00', -1);
--                                           ^^^  ← 원래 값 그대로

-- 2. 새 상태 INSERT
INSERT INTO inventory_v2 VALUES (1, 'Seoul', 80, now(), 1);
```

### 해결책 2: VersionedCollapsingMergeTree 사용 (순서 무관)

CollapsingMergeTree는 같은 파트 내에서도 sign=1이 sign=-1보다 **먼저** 와야 한다는 제약이 있다. VersionedCollapsingMergeTree는 이 순서 제약을 version 컬럼으로 해결한다.

```sql
CREATE TABLE inventory_v3 (
    item_id     UInt64,
    warehouse   LowCardinality(String),
    quantity    Int32,
    updated_at  DateTime,
    version     UInt64,    -- 단조 증가 버전
    sign        Int8
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY (item_id, warehouse);

-- version이 있으므로 취소 시 정확한 버전 지정
INSERT INTO inventory_v3 VALUES (1, 'Seoul', 100, '2024-01-01', 1, 1);  -- 원본
INSERT INTO inventory_v3 VALUES (1, 'Seoul', 100, '2024-01-01', 1, -1); -- 취소 (version=1 지정)
INSERT INTO inventory_v3 VALUES (1, 'Seoul', 80,  now(),        2, 1);  -- 새 상태 (version=2)
```

### 해결책 3: 기존 데이터 정합성 복구

```sql
-- 짝 없는 sign=-1 행 제거 (FINAL로 머지 후 확인)
OPTIMIZE TABLE inventory FINAL;

-- 정합성 확인
SELECT item_id, warehouse, sum(sign) AS net_sign
FROM inventory
GROUP BY item_id, warehouse
HAVING net_sign NOT IN (0, 1);
-- 결과가 나오면 아직 정합성 문제 존재
```

---

## 예방

```
CollapsingMergeTree 설계 체크리스트:

  ✅ ORDER BY는 변하지 않는 식별자만
  ✅ 상태 변경 시: (기존 값, sign=-1) → (새 값, sign=1) 순서 보장
  ✅ 조회 시 항상 sum(col * sign)으로 집계
  ✅ FINAL 또는 WHERE sign = 1 (머지 후) 패턴 인지

  동시성 높은 환경 → VersionedCollapsingMergeTree 사용
  단순 상태 대체   → ReplacingMergeTree가 더 적합
```

```sql
-- 올바른 조회 패턴
SELECT
    item_id,
    warehouse,
    sum(quantity * sign) AS current_quantity
FROM inventory
GROUP BY item_id, warehouse
HAVING sum(sign) > 0;   -- sign 합이 0인 것은 완전 상쇄됨
```
