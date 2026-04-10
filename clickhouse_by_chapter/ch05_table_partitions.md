# 5장. Table Partitions — 데이터 Lifecycle 관리의 단위

이 장에서는 **파티션(Partition)**을 다룬다. 4장에서 다룬 파트(Part)가 물리적 저장 단위라면, 파티션은 파트들을 **논리적으로 그룹화**하는 상위 개념이다. 파티셔닝은 강력한 기능이지만, 잘못 사용하면 오히려 성능을 저하시킬 수 있으므로 정확한 이해가 필수적이다.

**이 장의 핵심 메시지를 미리 밝힌다:**

> "Partitioning is primarily a data management technique and not a query optimization tool."
> — ClickHouse 공식 Best Practices

---

## 5.1 파티션이란 무엇인가

파티션은 MergeTree 엔진 패밀리에서 테이블의 데이터 파트들을 **특정 기준(시간 범위, 카테고리 등)**에 따라 조직화한 논리적 단위다.

`PARTITION BY` 절로 테이블 생성 시 정의한다:

```sql
CREATE TABLE uk.uk_price_paid_simple_partitioned
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = MergeTree
ORDER BY (town, street)
PARTITION BY toStartOfMonth(date);
```

이 테이블에서 `toStartOfMonth(date)` 결과값이 같은 행들은 동일 파티션에 속한다. 2020년 1월 데이터는 `2020-01-01` 파티션, 2020년 2월 데이터는 `2020-02-01` 파티션이 된다.

---

## 5.2 파티셔닝이 INSERT에 미치는 영향

파티셔닝이 **없는** 테이블에 INSERT하면 하나의 파트가 생성된다 (4장 참조).

파티셔닝이 **있는** 테이블에 INSERT하면, ClickHouse는 먼저 삽입할 행들을 파티션 키 값별로 분류한 뒤, **파티션마다 별도의 파트를 생성**한다.

```
INSERT 4행 (2020-01, 2020-01, 2020-03, 2020-03)
         │
         ▼
  ┌──────────────────────────────┐
  │ 파티션 키별 분류              │
  │  2020-01-01: 행1, 행2        │
  │  2020-03-01: 행3, 행4        │
  └──────────────────────────────┘
         │
         ▼
  파트 A (2020-01 파티션)   파트 B (2020-03 파티션)
  ① 정렬                    ① 정렬
  ② 컬럼 분리               ② 컬럼 분리
  ③ 압축                    ③ 압축
  ④ 디스크 기록              ④ 디스크 기록
```

하나의 INSERT가 여러 파티션에 걸친 데이터를 포함하면, **파티션 수만큼 파트가 생성**된다. 이것이 파티셔닝이 파트 수 증가를 유발하는 메커니즘이다.

### MinMax Index 자동 생성

파티셔닝이 활성화되면, ClickHouse는 각 파트에 대해 파티션 키에 사용된 컬럼의 **최솟값/최댓값 인덱스(MinMax Index)**를 자동 생성한다. 이 인덱스가 파티션 프루닝의 기반이 된다.

---

## 5.3 파티션 내에서만 머지된다

이것은 파티셔닝의 가장 중요한 동작 특성이다:

> **ClickHouse는 같은 파티션 내의 파트만 머지한다. 다른 파티션의 파트끼리는 절대 머지되지 않는다.**

이 규칙의 결과:

- 파티셔닝이 **없는** 테이블: 모든 파트가 머지 대상 → 시간이 지나면 소수의 큰 파트로 수렴
- 파티셔닝이 **있는** 테이블: 파티션별로 독립적으로 머지 → 파티션 수 × 파티션당 파트 수만큼의 파트가 유지

**여기서 고카디널리티 파티션 키의 위험이 발생한다.** 파티션이 수천 개면, 각 파티션 내의 파트들끼리만 머지할 수 있으므로 파트가 분산되어 전체 파트 수가 폭증한다. 이것이 "Too many parts" 에러의 주요 원인 중 하나다.

공식 문서의 권장:

> 파티션 키의 카디널리티는 **1,000 ~ 10,000 이하**로 유지하라.

---

## 5.4 파티션의 주요 용도: 데이터 관리

### TTL을 통한 자동 데이터 삭제

최근 12개월 데이터만 유지하는 시나리오:

```sql
CREATE TABLE uk.uk_price_paid_simple_partitioned
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = MergeTree
PARTITION BY toStartOfMonth(date)
ORDER BY (town, street)
TTL date + INTERVAL 12 MONTH DELETE;
```

월 단위 파티셔닝 + TTL 조합이면, 만료된 파티션의 전체 파트를 한 번에 제거할 수 있다. **행 단위 DELETE(mutation)와 달리 파트를 다시 쓰지 않으므로** 매우 효율적이다.

### Tiered Storage (Hot/Cold 분리)

오래된 데이터를 저렴한 스토리지로 자동 이동:

```sql
TTL date + INTERVAL 12 MONTH TO VOLUME 'slow_but_cheap';
```

SSD(hot) → S3/HDD(cold) 같은 계층형 스토리지 전략에서, 파티션 단위로 깔끔하게 데이터를 이동할 수 있다.

### 수동 파티션 관리

파티션 단위로 수행할 수 있는 주요 작업:

```sql
-- 특정 파티션 삭제 (즉시, 매우 빠름)
ALTER TABLE mytable DROP PARTITION '2023-01-01';

-- 파티션 분리 (테이블에서 떼어내기)
ALTER TABLE mytable DETACH PARTITION '2023-01-01';

-- 분리된 파티션 재연결
ALTER TABLE mytable ATTACH PARTITION '2023-01-01';

-- 파티션을 다른 테이블로 이동
ALTER TABLE source MOVE PARTITION '2023-01-01' TO TABLE archive;
```

이 작업들은 **메타데이터 수준**에서 수행되므로 행 단위 DELETE/INSERT보다 훨씬 빠르고 가볍다.

---

## 5.5 쿼리 최적화: 파티션 프루닝

파티션이 쿼리 성능을 도울 수 있는 **유일한 시나리오**는 다음 조건을 모두 만족할 때다:

1. 쿼리가 **소수의 파티션만** 대상으로 함 (이상적으로 1개)
2. 파티셔닝 키가 **프라이머리 키에 포함되지 않음**
3. WHERE 절에서 파티셔닝 키 컬럼으로 **필터링**

### 동작 원리: 3단계 프루닝

```sql
SELECT MAX(price) AS highest_price
FROM uk.uk_price_paid_simple_partitioned
WHERE date >= '2020-12-01'
  AND date <= '2020-12-31'
  AND town = 'LONDON';
```

이 쿼리에서 ClickHouse는 순차적으로 데이터를 좁혀간다:

**① MinMax Index 프루닝**: 각 파트의 `date` 컬럼 MinMax 인덱스를 검사하여, 쿼리 범위에 해당하지 않는 파트를 제외. 436개 파트 중 1개만 남김.

**② Partition 프루닝**: `toStartOfMonth(date)` 파티션 키 조건으로 추가 필터링.

**③ Primary Key 프루닝**: 남은 파트에서 `town` 컬럼의 sparse primary index로 granule 단위 필터링. 11개 granule 중 1개만 남김.

`EXPLAIN indexes = 1`로 확인 가능:

```sql
EXPLAIN indexes = 1
SELECT MAX(price) AS highest_price
FROM uk.uk_price_paid_simple_partitioned
WHERE date >= '2020-12-01'
  AND date <= '2020-12-31'
  AND town = 'LONDON';
```

결과에서 핵심 수치:
```
MinMax:     Parts: 1/436,   Granules: 11/3257
Partition:  Parts: 1/1,     Granules: 11/11
PrimaryKey: Parts: 1/1,     Granules: 1/11
```

436개 파트에서 1개 파트, 3257개 granule에서 1개 granule로 좁혀졌다. 최종적으로 8,192행만 스캔하여 6ms만에 결과를 반환한다.

---

## 5.6 파티셔닝의 역효과: 실측 비교

공식 문서에서 제공하는 **동일 데이터, 동일 쿼리**의 파티셔닝 유무 비교는 매우 중요하다.

### 쿼리: 파티션 키로 필터하지 않는 경우

```sql
SELECT MAX(price) AS highest_price
FROM uk.uk_price_paid_simple[_partitioned]
WHERE town = 'LONDON';
```

`town`은 프라이머리 키에는 포함되지만 파티션 키에는 포함되지 않으므로, 파티션 프루닝이 작동하지 않는다.

| 지표 | 파티셔닝 없는 테이블 | 파티셔닝 있는 테이블 |
|------|---------------------|---------------------|
| 활성 파트 수 | 1 (완전 머지됨) | 436 |
| 스캔 파트 수 | 1/1 | 431/436 |
| 스캔 granule 수 | 241/3,083 | 671/3,257 |
| 처리 행 수 | ~197만 | ~548만 |
| 소요 시간 | **12ms** | **90ms** |
| 피크 메모리 | 62 MiB | 163 MiB |

**파티셔닝된 테이블이 7.5배 느리고, 메모리를 2.6배 더 사용**한다.

이유:
- 파티셔닝으로 인해 데이터가 436개 파트에 분산 → 각 파트마다 sparse index가 독립적
- 비파티션 테이블은 하나의 큰 파트로 머지되어 primary index가 더 효과적으로 작동
- 431개 파트를 개별적으로 열고 index를 검사하는 오버헤드

이것이 공식 문서가 반복적으로 강조하는 메시지다:

> "Querying across all partitions is typically slower than running the same query on a non-partitioned table."

---

## 5.7 파티션 모니터링

### 파티션 목록 조회

```sql
-- 가상 컬럼으로 조회
SELECT DISTINCT _partition_value AS partition
FROM uk.uk_price_paid_simple_partitioned
ORDER BY partition ASC;

-- system.parts로 조회 (파티션별 파트 수, 행 수 포함)
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'uk')
  AND (`table` = 'uk_price_paid_simple_partitioned')
  AND active
GROUP BY partition
ORDER BY partition ASC;
```

### 파티션 건강 상태 모니터링

```sql
-- 파티션별 파트 수 이상 감지 (파트가 많은 파티션 찾기)
SELECT
    partition,
    count() AS parts,
    min(level) AS min_merge_level,
    max(level) AS max_merge_level
FROM system.parts
WHERE database = 'mydb'
  AND `table` = 'mytable'
  AND active
GROUP BY partition
HAVING parts > 10
ORDER BY parts DESC;
```

---

## 5.8 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **본질** | 데이터 관리(lifecycle) 기법이지, 쿼리 최적화의 주 수단이 아니다 |
| **INSERT 영향** | 파티션 키 값마다 별도 파트 생성 → 파트 수 증가 |
| **머지 제약** | 같은 파티션 내에서만 머지. 파티션 간 머지 불가 |
| **카디널리티 제한** | 파티션 키 카디널리티 1,000~10,000 이하 권장 |
| **주요 용도** | TTL 삭제, tiered storage, DROP/DETACH/MOVE PARTITION |
| **쿼리 최적화** | 파티션 키가 PK에 없고, 소수 파티션만 필터할 때만 유효 |
| **역효과** | 전체 파티션 스캔 시 비파티션 테이블보다 느림 (실측: 7.5배) |

---

## 5.9 운영 시 주의점

### 파티션 키 선택 원칙

1. **목적을 먼저 정하라**: "어떤 데이터 관리 작업을 할 것인가?"가 파티션 키 선택의 출발점이다. 쿼리 최적화 목적이라면 프라이머리 키나 projection을 먼저 고려하라.
2. **저카디널리티 유지**: 월 단위(`toStartOfMonth`), 주 단위(`toMonday`), 또는 카테고리 컬럼 등. 일(day) 단위는 수년간 데이터면 수천 개 파티션이 되므로 주의.
3. **쿼리 패턴과의 정합성**: 대부분의 쿼리가 특정 파티션만 대상으로 한다면 파티셔닝이 도움. 전체 스캔이 잦으면 오히려 해악.

### 흔한 실수

- **날짜 컬럼을 그대로 파티션 키로 사용** (`PARTITION BY date`): 날짜의 카디널리티가 매우 높아 파티션이 수천~수만 개 생성됨. `toStartOfMonth(date)` 같은 함수로 카디널리티를 줄여야 함.
- **파티셔닝 없이도 충분한데 파티셔닝 추가**: 프라이머리 키로 충분히 데이터를 좁힐 수 있다면 파티셔닝은 불필요한 복잡성을 추가할 뿐이다.
- **파티션 키를 프라이머리 키에도 포함**: 이 경우 파티션 프루닝의 쿼리 최적화 효과가 없다 (프라이머리 키만으로 이미 필터링 가능).

### "Too many parts"와 파티셔닝의 관계

파트 수 폭증의 가장 흔한 원인 조합:

```
과도한 파티셔닝 + 빈번한 소량 INSERT
= 파티션마다 작은 파트가 계속 생성
= 파티션 간 머지 불가
= 전체 파트 수 폭증
= parts_to_throw_insert 초과
= INSERT 거부
```

진단 쿼리:

```sql
SELECT
    count() AS total_active_parts,
    countDistinct(partition) AS total_partitions,
    total_active_parts / total_partitions AS avg_parts_per_partition
FROM system.parts
WHERE database = 'mydb'
  AND `table` = 'mytable'
  AND active;
```

`total_partitions`이 수천 이상이면 파티션 키 재검토가 필요하다.

---

*다음 장: 6장. Table Part Merges — 백그라운드 최적화의 핵심*
