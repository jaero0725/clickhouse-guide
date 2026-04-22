# 7장. Primary Index — Sparse Index의 원리와 ORDER BY 키 설계

ClickHouse의 프라이머리 인덱스는 전통적 RDBMS의 B-Tree 인덱스와 **근본적으로 다르다**. 이 차이를 이해하지 못하면 키 설계에서 오류를 범하고, 기대한 쿼리 성능을 얻지 못한다. 이 장에서는 sparse index의 구조, 동작 원리, 그리고 ORDER BY 키 설계 전략을 다룬다.

---

## 7.1 Sparse Index란 무엇인가

### 전통적 인덱스와의 비교

| | B-Tree (PostgreSQL 등) | Sparse Index (ClickHouse) |
|---|----------------------|--------------------------|
| **인덱싱 단위** | 모든 행 (row-level) | N번째 행마다 (granule-level) |
| **메모리 소비** | 행 수에 비례 (대용량 시 디스크 사용) | 행 수 / 8192에 비례 (항상 메모리 상주 가능) |
| **유니크 제약** | 지원 | **미지원** (같은 키 값의 중복 행 허용) |
| **Point query** | O(log n) 후 직접 행 접근 | O(log n) 후 granule 전체 읽기 |
| **Range scan** | 효율적 | 매우 효율적 (연속된 granule skip) |

공식 문서의 핵심 설명:

> "The primary key itself is 'sparse'. It does not address every single row, but only some ranges of data."

### Granule: 데이터 처리의 최소 단위

ClickHouse는 각 컬럼의 데이터를 논리적으로 **granule** 단위로 분할한다. 기본적으로 하나의 granule은 **8,192행**이며, 이것이 ClickHouse가 처리하는 가장 작은 데이터 단위다.

```
Column data:
  [granule 0: row 0~8191] [granule 1: row 8192~16383] [granule 2: row 16384~24575] ...
```

Sparse index는 **각 granule의 첫 번째 행의 프라이머리 키 값**만 저장한다. 따라서 인덱스 엔트리 수 = 전체 행 수 / `index_granularity`.

---

## 7.2 Sparse Index의 생성 과정

예시 테이블:

```sql
CREATE TABLE uk.uk_price_paid_simple
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = MergeTree
ORDER BY (town, street);
```

INSERT 시 다음이 순서대로 일어난다:

1. 삽입된 행들을 **sorting key `(town, street)` 순서로 정렬**
2. 각 컬럼의 데이터를 별도 파일로 분리, 압축, 디스크에 저장
3. 정렬된 데이터를 논리적으로 **8,192행 단위의 granule**로 분할
4. 각 granule의 **첫 번째 행의 (town, street) 값**을 `primary.idx`에 기록

결과적으로 primary.idx의 내용은 다음과 같은 형태:

```
Entry 1:  (ABBOTS LANGLEY, ABBEY DRIVE)        → granule 0
Entry 2:  (ABERDARE, RICHARDS TERRACE)         → granule 1
Entry 3:  (ABERGELE, PEN Y CAE)                → granule 2
Entry 4:  (ABINGDON, CHAMBRAI CLOSE)           → granule 3
...
```

### 각 파트가 독립적인 인덱스를 가진다

4장에서 배운 대로 테이블은 여러 파트로 구성된다. **각 파트는 자체 primary index를 가진다**. 쿼리 실행 시 모든 파트의 인덱스가 독립적으로 사용된다.

---

## 7.3 쿼리 시 Index 사용 과정

```sql
SELECT max(price)
FROM uk.uk_price_paid_simple
WHERE town = 'LONDON' AND street = 'OXFORD STREET';
```

이 쿼리의 실행 과정:

**① Index 로드**: 테이블의 모든 활성 파트의 primary index를 메모리에 로드 (이미 상주해 있음)

**② Granule 선별**: 인덱스 엔트리를 스캔하여 `town = 'LONDON' AND street = 'OXFORD STREET'` 조건에 **매칭될 수 없는 granule을 제외**. 이진 탐색으로 후보 범위를 빠르게 좁힘

**③ Granule 로드 + 처리**: 선별된 granule만 디스크에서 읽어 압축 해제, 실제 필터링 및 집계 수행

`EXPLAIN indexes = 1`로 이 과정을 관찰할 수 있다:

```sql
EXPLAIN indexes = 1
SELECT max(price)
FROM uk.uk_price_paid_simple
WHERE town = 'LONDON' AND street = 'OXFORD STREET';

         PrimaryKey
           Keys:
             town
             street
           Condition: and((street in ['OXFORD STREET', 'OXFORD STREET']),
                          (town in ['LONDON', 'LONDON']))
           Parts: 3/3
           Granules: 3/3609
```

3,609개 granule 중 **3개만 선택**되었다. 나머지 3,606개는 인덱스 분석만으로 skip되어 디스크에서 읽히지도 않는다.

실제 실행 결과:

```
1 row in set. Elapsed: 0.010 sec.
Processed 24.58 thousand rows, 159.04 KB
(약 2,955만 행 중 2.5만 행만 처리 → 99.9% skip)
```

---

## 7.4 Sparse Index의 트레이드오프

### 장점: 수조 행에서도 미미한 메모리 소비

공식 architecture.md:

> "We made the index sparse because we must be able to maintain trillions of rows per single server without noticeable memory consumption for the index."

1조 행 / 8,192 = 약 1.2억 인덱스 엔트리. 키가 (UInt64, Date) = 10 바이트라면 약 1.2GB. B-Tree로 같은 데이터를 인덱싱하면 TB 단위가 필요하다.

### 단점: Point Query의 최소 읽기량

단일 행을 찾더라도 **최소 1 granule(8,192행)**을 읽어야 하고, 해당 granule이 속한 **압축 블록 전체를 해제**해야 한다.

> "ClickHouse is not suitable for a high load of simple point queries, because the entire range with index_granularity rows must be read for each key, and the entire compressed block must be decompressed for each column."

### 단점: 유니크 제약 불가

Sparse index는 모든 행을 인덱싱하지 않으므로 INSERT 시 키의 존재 여부를 확인할 수 없다. 같은 키 값을 가진 행이 테이블에 여러 개 존재할 수 있다.

---

## 7.5 ORDER BY 키 설계 전략

공식 best-practices의 핵심 원칙:

> "When selecting an ordering key, prioritize columns frequently used in query filters (i.e. the WHERE clause), especially those that exclude large numbers of rows."

### 기본 규칙

**규칙 1: WHERE 절에서 자주 사용되는 컬럼 우선**

가장 중요한 원칙이다. 인덱스가 효과를 발휘하려면 쿼리의 필터 조건이 ordering key 컬럼과 일치해야 한다.

**규칙 2: 저카디널리티 컬럼을 먼저, 고카디널리티 컬럼을 뒤에**

```
좋은 예:  ORDER BY (PostTypeId, toDate(CreationDate))
          카디널리티 8       카디널리티 ~10,000
```

저카디널리티 컬럼이 앞에 오면 인덱스의 첫 번째 키만으로도 대량의 granule을 skip할 수 있다. 고카디널리티 컬럼은 범위를 세밀하게 좁히는 보조 역할을 한다.

**규칙 3: 상관관계 높은 컬럼을 포함하면 압축률 향상**

ordering key로 정렬하면 **모든 컬럼**이 해당 순서로 정렬된다 (키에 포함되지 않은 컬럼도). 유사한 값이 물리적으로 연속 배치되므로 압축이 효과적이다.

**규칙 4: 4~5개 키면 충분**

너무 많은 키는 인덱스 크기를 키우고, 정렬 비용을 높이며, 실질적 효과가 줄어든다.

**규칙 5: 날짜는 정밀도를 낮춰라**

DateTime 대신 `toDate(CreationDate)`를 사용하면 인덱스 엔트리 크기가 8바이트에서 2바이트로 줄어들고, 필터링 속도도 빨라진다. 일(day) 단위 필터링으로 충분한 경우가 대부분이다.

### 실측 효과

같은 데이터, 같은 쿼리로 비교:

```sql
-- ORDER BY 없음 (ORDER BY tuple())
SELECT count() FROM posts_unordered
WHERE CreationDate >= '2024-01-01' AND PostTypeId = 'Question';
-- Processed 59.82 million rows, 361.34 MB
-- Elapsed: 0.055 sec (전체 스캔)

-- ORDER BY (PostTypeId, toDate(CreationDate))
SELECT count() FROM posts_ordered
WHERE CreationDate >= '2024-01-01' AND PostTypeId = 'Question';
-- Processed 196.53 thousand rows, 1.77 MB
-- Elapsed: 0.013 sec (Granules: 39/7578)
```

읽기 데이터량이 **300배 이상 감소**하고, 실행 시간은 **4배 이상 빨라졌다**.

### 키 선택 시 흔한 실수

**실수 1: 고카디널리티 컬럼을 첫 번째에 배치**

```sql
-- 나쁜 예: user_id (수백만 고유값)가 첫 번째
ORDER BY (user_id, event_type, date)

-- 좋은 예: event_type (수십 고유값)가 첫 번째
ORDER BY (event_type, date, user_id)
```

**실수 2: 쿼리 패턴과 무관한 키 선택**

INSERT 순서나 데이터 모델의 논리적 순서가 아닌, **실제 쿼리의 WHERE 절 패턴**에 맞춰야 한다.

**실수 3: 생성 후 변경 불가를 간과**

```
ORDER BY 키는 테이블 생성 시에만 정의된다.
나중에 추가하거나 변경할 수 없다.
```

대안으로 **Projection**을 사용할 수 있지만, 이는 데이터 복제를 발생시킨다.

---

## 7.6 Ordering Key vs Primary Key

공식 문서에서 이 둘을 혼용하지만, 엄밀하게는 다르다:

- **Ordering Key** (`ORDER BY`): 디스크 상 데이터의 물리적 정렬 순서를 정의
- **Primary Key** (`PRIMARY KEY`): sparse index에 저장되는 컬럼을 정의

대부분의 경우 둘은 동일하며, `PRIMARY KEY`를 별도로 지정하지 않으면 `ORDER BY` 키가 자동으로 primary key가 된다.

별도 지정이 필요한 경우: ORDER BY 키의 앞부분만 인덱싱하고 싶을 때.

```sql
-- ORDER BY는 (a, b, c)이지만, index는 (a, b)만
CREATE TABLE t (...)
ENGINE = MergeTree
ORDER BY (a, b, c)
PRIMARY KEY (a, b);
```

이 경우 데이터는 (a, b, c)로 정렬되지만, sparse index는 (a, b)만 저장해 인덱스 크기를 줄인다.

---

## 7.7 인덱스 내용 직접 확인: mergeTreeIndex

```sql
-- 파트별 인덱스 엔트리 수 확인
SELECT
    part_name,
    max(mark_number) AS entries
FROM mergeTreeIndex('uk', 'uk_price_paid_simple')
GROUP BY part_name;

-- 실제 인덱스 엔트리 내용 확인
SELECT
    mark_number + 1 AS entry,
    town,
    street
FROM mergeTreeIndex('uk', 'uk_price_paid_simple')
WHERE part_name = (SELECT any(part_name)
                   FROM mergeTreeIndex('uk', 'uk_price_paid_simple'))
ORDER BY mark_number ASC
LIMIT 10;

    ┌─entry─┬─town───────────┬─street───────────┐
 1. │     1 │ ABBOTS LANGLEY │ ABBEY DRIVE      │
 2. │     2 │ ABERDARE       │ RICHARDS TERRACE │
 3. │     3 │ ABERGELE       │ PEN Y CAE        │
 ...
```

이 쿼리로 인덱스의 실제 분포를 확인하여, granule이 얼마나 효과적으로 구분되는지 판단할 수 있다.

---

## 7.8 index_granularity 튜닝

기본값 8,192는 대부분의 워크로드에 적합하지만, 극단적인 경우 조정을 고려할 수 있다.

| 조정 방향 | 효과 | 트레이드오프 |
|-----------|------|-------------|
| **줄이기** (예: 1024) | granule이 작아져 skip 정밀도 향상 | 인덱스 엔트리 8배 증가 → 메모리 소비 증가, mark 파일 크기 증가 |
| **늘리기** (예: 65536) | 인덱스 크기 감소, mark 파일 축소 | skip 정밀도 하락, point query 최소 읽기량 증가 |

일반적으로 기본값을 유지하되, 극히 넓은 테이블(수백 컬럼)이나 극히 좁은 테이블(2~3 컬럼)에서만 조정을 검토하라.

---

## 7.9 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Sparse index** | granule(8,192행)의 첫 행만 인덱싱. 메모리 상주 가능. 수조 행 지원 |
| **동작 원리** | 인덱스 스캔 → 매칭 불가능한 granule skip → 남은 granule만 로드·처리 |
| **각 파트가 독립 인덱스** | 파트가 많을수록 인덱스도 분산됨 (파티셔닝 과다 시 비효율) |
| **Point query 한계** | 최소 1 granule + 1 압축 블록 읽기 필수 |
| **유니크 제약 없음** | 중복 키 행 허용. INSERT 시 존재 확인 불가 |
| **키 설계 원칙** | WHERE 절 빈도 → 저카디널리티 우선 → 4~5개 키 → 날짜 정밀도 낮추기 |
| **생성 후 변경 불가** | ORDER BY 키는 테이블 생성 시 확정. Projection이 대안 (데이터 복제) |

---

## 7.10 운영 시 주의점

### 키 설계 전 쿼리 패턴 분석

```sql
-- 가장 빈번한 쿼리의 WHERE 절 패턴 확인
SELECT
    normalized_query_hash,
    count() AS query_count,
    any(query) AS sample_query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND event_date >= today() - 7
GROUP BY normalized_query_hash
ORDER BY query_count DESC
LIMIT 20;
```

### EXPLAIN으로 인덱스 효과 측정

```sql
EXPLAIN indexes = 1
SELECT ...
FROM mytable
WHERE ...;

-- 확인 포인트:
-- Granules: X/Y  → X가 Y 대비 충분히 작은지?
-- X/Y 비율이 높으면(skip이 적으면) 키 재설계 고려
```

### 인덱스가 안 먹히는 패턴

- **키의 앞부분을 건너뛴 필터**: `ORDER BY (a, b, c)`에서 `WHERE b = ...`만 필터하면 인덱스 효과 미미. a를 먼저 필터해야 b의 인덱스가 효과적.
- **함수 적용**: `WHERE toYear(date) = 2024`는 인덱스를 사용할 수 있지만, `WHERE transform(col) = ...` 같은 복잡한 함수는 인덱스를 우회할 수 있다.
- **부정 조건**: `WHERE col != value`는 대부분의 granule을 skip하지 못한다.

### Data Skipping Index 보조 활용

프라이머리 인덱스로 커버하지 못하는 컬럼에 대해 보조적으로 skipping index를 추가할 수 있다:

```sql
ALTER TABLE mytable ADD INDEX idx_name col_name TYPE minmax GRANULARITY 4;
```

단, 프라이머리 인덱스의 대체가 아닌 **보조 수단**이다. 효과는 데이터 분포에 크게 의존한다.

---

*다음 장: 8장. Shards와 Replicas — 분산 아키텍처*

---

## 7.11 시각적 이해 — ORDER BY 키 설계가 성능에 미치는 영향

동일 데이터, 동일 쿼리에서 ORDER BY 키 순서에 따른 차이.

### 시나리오: 로그인 이벤트 테이블

```
저카디널리티: status (약 4개), country (약 200개)
고카디널리티: user_id (수백만), event_time (수억 유니크)

주요 쿼리: WHERE status = 'failed' AND event_time >= '...'
```

### 설계 A — 고카디널리티 먼저 (나쁨)

```sql
ORDER BY (user_id, event_time, status, country)
```

```
파트 내부 레이아웃:
 user_id=1,       event_time=09:00, status=ok,   country=KR
 user_id=1,       event_time=09:01, status=ok,   country=KR
  ...
 user_id=2,       event_time=09:00, status=fail, country=US
  ...
 user_id=1000000, event_time=23:59, status=ok,   country=JP

WHERE status = 'failed' AND event_time >= '...'
  → user_id 필터 없으면 인덱스 이진 탐색 활용 불가
  → 전체 파트 스캔 (풀 스캔)
```

### 설계 B — 저카디널리티 먼저 (좋음)

```sql
ORDER BY (status, toDate(event_time), country, user_id)
```

```
파트 내부 레이아웃:
 status=fail, date=2024-04-01, country=JP, user_id=...
  (수만 건)
 status=fail, date=2024-04-02, country=KR, user_id=...
  (수만 건)   ← 쿼리가 요구하는 범위
 status=ok,   date=2024-04-01, country=JP, user_id=...
  (수억 건)   ← 인덱스로 바로 skip
 status=ok,   date=2024-04-30, country=US, user_id=...

WHERE status = 'failed' AND event_time >= '2024-04-01'
  → 인덱스 이진 탐색으로 status=fail 구간에 점프
  → 날짜 조건으로 추가 granule 필터링
  → 실제로 읽는 데이터 1% 미만
```

### 실측 비교

| 지표 | 설계 A | 설계 B |
|------|--------|--------|
| 스캔 granule | 3,609 / 3,609 (100%) | 3 / 3,609 (0.08%) |
| 읽은 행 | 2,950만 | 24,580 |
| 소요 시간 | 8,500ms | 10ms |

**850배 차이.** 단지 키 순서만 바꿨을 뿐.

---

## 7.12 시각적 이해 — Granule Skip의 원리

```sql
CREATE TABLE events (...)
ORDER BY (status, date);
```

INSERT 후 디스크 레이아웃:

```
granule 0 [0 ~ 8191]:     status=fail, date=2024-04-01 ~ 2024-04-03
granule 1 [8192 ~ 16383]: status=fail, date=2024-04-04 ~ 2024-04-08
granule 2 [16384 ~ ...]:  status=ok,   date=2024-04-01 ~ 2024-04-02
granule 3:                status=ok,   date=2024-04-03 ~ 2024-04-05
granule 4:                status=ok,   date=2024-04-06 ~ 2024-04-09
...

primary.idx (각 granule의 첫 행 키 저장):
[0]: (fail, 2024-04-01)
[1]: (fail, 2024-04-04)
[2]: (ok,   2024-04-01)
[3]: (ok,   2024-04-03)
[4]: (ok,   2024-04-06)
```

### 쿼리 실행 시

```sql
SELECT count() FROM events 
WHERE status = 'ok' AND date >= '2024-04-03' AND date <= '2024-04-07';
```

```
1. primary.idx를 이진 탐색

2. 조건 매칭:
   granule 0 (fail): status 불일치 → skip
   granule 1 (fail): status 불일치 → skip
   granule 2 (ok, 2024-04-01~02): date 조건 미포함 → skip
   granule 3 (ok, 2024-04-03~05): 겹침! → 읽기
   granule 4 (ok, 2024-04-06~09): 겹침! → 읽기

3. 선택된 granule만 디스크에서 읽기
   → 원본 데이터의 일부만 I/O 발생
```

### "인덱스가 안 먹히는" 패턴

```sql
-- 나쁨: 키의 앞부분(status)을 안 쓰고 date만 필터
WHERE date = '2024-04-05'
  → status 값이 섞여 있어 granule skip 힘듦

-- 나쁨: 함수 래핑 (일부는 작동, 복잡한 건 안 됨)
WHERE transform(status) = 'failed'
  → 최적화기가 인덱스 못 씀

-- 나쁨: 부정 조건
WHERE status != 'ok'
  → 대부분 granule이 매칭 → skip 효과 없음

-- 좋음: 키 순서대로 필터
WHERE status = 'failed' AND date >= '...'

-- 좋음: IN 절
WHERE status IN ('failed', 'timeout')
  → 여러 범위를 각각 이진 탐색
```
