# 4장. Table Parts — MergeTree 스토리지의 물리적 구조

이 장에서는 ClickHouse의 가장 핵심적인 개념인 **Table Part(테이블 파트)**를 다룬다. 파트를 이해하면 MergeTree의 INSERT, SELECT, 머지, 장애 복구의 동작 원리가 자연스럽게 따라온다.

---

## 4.1 파트란 무엇인가

MergeTree 엔진 패밀리의 모든 테이블에서 데이터는 디스크 위에 **immutable한 data part들의 집합**으로 조직된다. 파트는 ClickHouse 스토리지의 최소 물리 단위이다.

공식 문서의 정의:

> "The data from each table in the ClickHouse MergeTree engine family is organized on disk as a collection of immutable data parts."

핵심 키워드는 **immutable**이다. 파트는 생성되고 삭제될 수는 있지만, 한 번 생성된 파트의 내용은 **절대 수정되지 않는다**. 이 불변성이 ClickHouse의 동시성 모델, 장애 복구, 스냅샷 기반 SELECT의 기반이다.

---

## 4.2 파트의 생성 과정: INSERT의 4단계

INSERT가 실행되면 ClickHouse 서버는 다음 4단계를 순차적으로 수행하여 새 파트를 생성한다.

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

### ① Sorting — 정렬

삽입된 행들을 테이블의 **sorting key** `(town, street)` 순서로 정렬한다. 동시에 정렬된 행에 대한 **sparse primary index**를 생성한다.

이 정렬 단계가 중요한 이유: 데이터가 정렬된 상태로 저장되므로 쿼리 시 이진 탐색으로 범위를 좁힐 수 있고, 동일 값이 연속 배치되어 압축률이 향상된다.

클라이언트가 데이터를 프라이머리 키 순서로 **사전 정렬**하여 보내면, 서버가 이 정렬 단계를 skip할 수 있어 INSERT 성능이 향상된다.

### ② Splitting — 컬럼 분리

정렬된 데이터를 **컬럼별로 분리**한다. 컬럼 지향 저장의 핵심 단계다.

### ③ Compression — 압축

각 컬럼을 독립적으로 압축한다. 컬럼별로 데이터 타입과 분포가 균일하므로, 범용 압축(ZSTD, LZ4)이나 특화 코덱(Delta, Gorilla, FPC 등)이 높은 압축률을 달성한다.

### ④ Writing to Disk — 디스크 기록

압축된 컬럼들을 **바이너리 컬럼 파일**로 새 디렉터리에 저장한다. 이 디렉터리 하나가 곧 하나의 data part이다. Sparse primary index도 압축되어 같은 디렉터리에 저장된다.

---

## 4.3 파트의 물리적 구조: 디렉터리 내부

하나의 파트 디렉터리에는 다음 파일들이 포함된다:

### 컬럼 데이터 파일 (`column.bin`)

모든 테이블 컬럼이 별도의 `column.bin` 파일로 저장된다. 파일 내부는 **압축 블록(compressed block)**들의 연속이다.

공식 architecture.md에서 설명하는 압축 블록의 구조:

- 각 블록은 비압축 기준 보통 **64KB ~ 1MB** 크기
- 블록 내에서 컬럼 값들은 프라이머리 키 순서대로 연속 배치
- 모든 컬럼에서 같은 행 순서가 유지됨 → 여러 컬럼을 동시에 순회할 때 대응하는 행의 값을 얻을 수 있음

### Primary Index 파일 (`primary.idx`)

N번째 행마다 프라이머리 키 값을 저장하는 sparse index. N = `index_granularity` (기본값 8192).

공식 문서 원문:

> "A separate primary.idx file has the value of the primary key for each N-th row, where N is called index_granularity (usually, N = 8192)."

`primary.idx` 데이터는 **항상 메모리에 상주**한다. 수조 행이 있어도 sparse하기 때문에 메모리 소비가 미미하다.

### Mark 파일 (`column.mrk`)

각 컬럼마다 존재하며, N번째 행에 대한 **오프셋 쌍**을 저장한다:

```
mark = (파일 내 압축 블록 시작 오프셋, 압축 해제된 블록 내 데이터 시작 오프셋)
```

보통 압축 블록이 mark에 정렬되어 있어 두 번째 오프셋은 0인 경우가 많다. mark 파일은 **캐시**된다.

### 추가 메타데이터

파트는 **자기 완결적(self-contained)**이다. 중앙 카탈로그 없이 파트의 내용을 해석할 수 있는 모든 메타데이터를 포함한다:

- Secondary data skipping indexes
- Column statistics
- Checksums
- Min-max indexes (파티셔닝 사용 시)
- 기타 메타데이터

---

## 4.4 쿼리 시 파트 읽기 과정

SELECT 쿼리가 파트에서 데이터를 읽는 과정:

```
1. primary.idx 검사
   → 요청 데이터가 포함될 수 있는 granule 범위 식별 (이진 탐색)

2. column.mrk 참조
   → 해당 granule 범위의 시작 오프셋 계산

3. column.bin 읽기
   → 압축 블록 로드 → 압축 해제 → 데이터 추출
```

**Sparse index의 대가**: index_granularity 행 단위로만 범위를 좁힐 수 있으므로, 단일 키 조회(point query)에도 최대 8,192개 행을 읽고 전체 압축 블록을 해제해야 한다. 이것이 ClickHouse가 **대량의 단순 point query에 적합하지 않은 이유**다.

공식 문서의 설명:

> "ClickHouse is not suitable for a high load of simple point queries, because the entire range with index_granularity rows must be read for each key, and the entire compressed block must be decompressed for each column."

### SELECT의 스냅샷 격리

SELECT가 실행되면 **쿼리 시작 시점의 파트 집합(스냅샷)**을 기준으로 동작한다. 쿼리 실행 도중 새로운 INSERT나 머지가 발생해도 SELECT에 영향을 주지 않는다. 파트가 immutable하기 때문에 가능한 모델이다.

---

## 4.5 MergeTree ≠ LSM Tree

이 구분은 매우 중요하다. 공식 architecture.md에서 명확히 밝힌다:

> "MergeTree is not an LSM tree because it does not contain MEMTABLE and LOG: inserted data is written directly to the filesystem."

| | LSM Tree | MergeTree |
|---|---------|-----------|
| **메모리 버퍼** | MEMTABLE (인메모리 쓰기 버퍼) | 없음 |
| **WAL** | Write-Ahead Log | 없음 |
| **INSERT 경로** | MEMTABLE → WAL → 플러시 → 디스크 | 즉시 디스크 파트 생성 |
| **최적 사용** | 빈번한 소량 쓰기 | 배치 단위 대량 쓰기 |

MergeTree가 MEMTABLE을 두지 않는 이유는 단순성을 위해서이며, 이미 배치 단위 INSERT를 전제로 설계되었기 때문이다.

**소량 빈번 INSERT가 문제인 이유**: INSERT마다 새 파트(디렉터리 + 파일들)가 생성된다. 초당 수천 번 INSERT하면 수천 개의 파트가 쌓이고, 백그라운드 머지가 따라잡지 못해 "Too many parts" 에러가 발생한다.

**대응 방법**: 
- 클라이언트에서 배치로 모아 INSERT (권장: 최소 20,000행 이상)
- **Async insert 모드**: ClickHouse가 여러 소량 INSERT를 서버 측에서 버퍼링하여 하나의 파트로 생성. 버퍼 크기 임계값 또는 타임아웃 초과 시 파트 생성

---

## 4.6 파트의 생명주기

파트는 다음과 같은 생명주기를 거친다:

```
① INSERT → 새 파트 생성 (active, level=0)
     │
② 백그라운드 머지 → 여러 작은 파트가 하나의 큰 파트로 병합
     │                    새 파트 생성 (active, level=N+1)
     │                    원본 파트들은 inactive로 마킹
     │
③ 일정 시간 경과 → inactive 파트 삭제
     │                (old_parts_lifetime 설정)
     │
④ 장애 복구 시 → inactive 원본 파트로 롤백 가능
```

머지된 파트가 손상되었을 때, 원본 파트가 아직 남아 있으면 이를 사용해 복구할 수 있다. 이것이 머지 후에도 원본 파트를 일정 기간 보관하는 이유다.

### 파트 이름의 의미

파트 디렉터리 이름은 `all_0_5_1` 같은 형식이다:

```
all_0_5_1
 │  │ │ │
 │  │ │ └── merge level (1 = 한 번 머지됨)
 │  │ └──── max block number
 │  └────── min block number
 └───────── partition name ("all" = 파티셔닝 미사용)
```

- **level 0**: 새로 INSERT된 파트 (머지되지 않음)
- **level 1 이상**: 머지된 파트. 레벨이 높을수록 더 많은 머지를 거침
- **min/max block number**: 이 파트에 포함된 원본 블록 번호 범위

---

## 4.7 파트의 크기 제한과 머지 목표

백그라운드 머지는 파트들을 결합하여 **설정 가능한 최대 압축 크기**(기본 약 150GB)까지 키운다.

```
max_bytes_to_merge_at_max_space_in_pool
```

이 값에 도달한 파트는 더 이상 머지 대상이 되지 않는다. 이것이 시간이 지남에 따라 파트 수가 안정화되는 메커니즘이다.

---

## 4.8 Write Amplification

머지는 필연적으로 **write amplification(쓰기 증폭)**을 발생시킨다. 공식 문서:

> "Of course, merging leads to 'write amplification'."

동일한 데이터가 INSERT 시 한 번, 머지 시 다시 한 번(이상) 디스크에 쓰인다. 이는 컬럼 지향 스토리지에서 읽기 성능을 최적화하기 위한 의도적 트레이드오프다.

Write amplification은 다음과 같은 영향을 미친다:
- 디스크 I/O 대역폭 소비 (특히 SSD 수명에 영향)
- 머지 중 일시적으로 디스크 공간이 원본 + 결과 파트 양쪽 필요
- 머지가 CPU와 I/O 자원을 소비하므로 쿼리 성능에 간접적 영향

---

## 4.9 모니터링: 파트 현황 확인

### 방법 1: `_part` 가상 컬럼

```sql
SELECT _part
FROM uk.uk_price_paid_simple
GROUP BY _part
ORDER BY _part ASC;

   ┌─_part───────┐
1. │ all_0_5_1   │
2. │ all_12_17_1 │
3. │ all_18_23_1 │
4. │ all_6_11_1  │
   └─────────────┘
```

### 방법 2: `system.parts` 시스템 테이블

```sql
SELECT
    name,
    level,
    rows,
    bytes_on_disk,
    modification_time
FROM system.parts
WHERE (database = 'uk')
  AND (`table` = 'uk_price_paid_simple')
  AND active
ORDER BY name ASC;
```

`level` 컬럼이 핵심이다:
- **level = 0**: INSERT로 직접 생성된 파트
- **level ≥ 1**: 머지를 거친 파트. 숫자가 클수록 더 많은 머지를 거침

### 방법 3: 파트 수 추이 모니터링

```sql
-- 테이블별 활성 파트 수 확인 (Too many parts 위험 감지)
SELECT
    database,
    `table`,
    count() AS active_parts,
    sum(rows) AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS total_size
FROM system.parts
WHERE active
GROUP BY database, `table`
ORDER BY active_parts DESC;
```

---

## 4.10 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **파트의 정의** | MergeTree 스토리지의 최소 물리 단위. 디스크 상의 하나의 디렉터리 |
| **Immutability** | 파트는 생성·삭제만 가능. 수정 불가. 동시성과 장애 복구의 기반 |
| **INSERT → 파트 생성** | 정렬 → 컬럼 분리 → 압축 → 디스크 기록의 4단계 |
| **물리 구조** | column.bin (압축 블록), primary.idx (sparse index, 항상 메모리), column.mrk (오프셋 마크) |
| **Self-contained** | 중앙 카탈로그 없이 파트만으로 데이터 해석 가능 |
| **MergeTree ≠ LSM Tree** | MEMTABLE/WAL 없음. 디스크에 직접 기록. 배치 INSERT에 최적화 |
| **스냅샷 격리** | SELECT는 쿼리 시작 시점의 파트 집합을 기준으로 실행 |
| **Write Amplification** | 머지로 인한 필연적 트레이드오프. 읽기 최적화를 위한 대가 |

---

## 4.11 운영 시 주의점

### INSERT 전략

- **배치 크기**: 최소 20,000행 이상을 권장. 이상적으로는 수만~수십만 행 단위
- **Async insert**: 소량 빈번 INSERT가 불가피하면 `async_insert = 1` 활성화. 서버가 자동으로 배칭
- **사전 정렬**: 클라이언트에서 프라이머리 키 순서로 데이터를 정렬하여 보내면 서버의 정렬 단계가 skip되어 INSERT 성능 향상

### "Too many parts" 방지

이 에러는 파트 수가 `parts_to_throw_insert` (기본 3000) 임계값을 초과하면 발생한다. INSERT가 거부되므로 데이터 유입이 중단된다.

원인 진단:
```sql
-- 파티션별 파트 수 확인
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows,
    max(modification_time) AS latest_insert
FROM system.parts
WHERE database = 'mydb' AND `table` = 'mytable' AND active
GROUP BY partition
ORDER BY parts DESC;
```

주요 원인들:
- 소량 빈번 INSERT (배칭 없이)
- 과도한 파티셔닝 (파티션 수가 너무 많으면 파티션 내 머지만 가능하므로 파트가 분산)
- 머지가 따라잡지 못할 정도의 INSERT 속도
- 디스크 I/O 병목으로 머지 지연

### 디스크 공간 관리

- 머지 중에는 원본 파트 + 결과 파트가 동시에 존재하므로, 일시적으로 테이블 크기의 약 **2배 디스크 공간**이 필요할 수 있다
- `old_parts_lifetime` (기본 480초 = 8분): 머지 후 inactive 파트 보관 시간. 장애 복구에 필요하지만, 디스크가 부족하면 줄일 수 있다
- `system.parts`에서 `active = 0`인 파트의 용량을 모니터링하라

### Point Query 성능

ClickHouse의 sparse index 특성상 단일 키 조회는 최소 `index_granularity` (8192) 행을 읽는다. Point query가 주 워크로드라면:
- ClickHouse가 적합한 도구인지 재고하라 (OLTP DB가 더 적합할 수 있음)
- 불가피하다면 `index_granularity`를 줄이는 것을 고려하되, 인덱스 메모리 소비와 트레이드오프를 이해하라

---

*다음 장: 5장. Table Partitions — 데이터 lifecycle 관리의 단위*
