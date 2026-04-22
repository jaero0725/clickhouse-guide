# 1장. ClickHouse 개요와 설계 철학

## 1.1 ClickHouse는 어떤 데이터베이스인가

ClickHouse는 OLAP(Online Analytical Processing) 워크로드에 특화된 **컬럼 지향(column-oriented) DBMS**이다. 공식 문서의 첫 문장이 이를 명확히 한다:

> "ClickHouse is a true column-oriented DBMS. Data is stored by columns, and during the execution of arrays (vectors or chunks of columns)."

"true column-oriented"라는 표현에 주목할 필요가 있다. 단순히 데이터를 컬럼 단위로 저장하는 것을 넘어서, **쿼리 실행 자체가 컬럼(벡터) 단위로 이루어진다**는 뜻이다. 이것이 ClickHouse의 성능을 결정짓는 핵심 설계 원칙이다.

---

## 1.2 OLAP vs OLTP: 근본적인 트레이드오프

모든 DBMS는 OLAP과 OLTP 사이의 스펙트럼 위에 존재한다. ClickHouse는 이 스펙트럼에서 OLAP 쪽 극단에 위치한다.

**OLTP (Online Transactional Processing)**는 지속적인 트랜잭션 스트림을 처리하며 데이터의 현재 상태를 끊임없이 수정한다. 행(row) 단위 연산에 최적화되어 있으며, 개별 행의 읽기/쓰기가 빨라야 한다.

**OLAP (Online Analytical Processing)**는 대량의 과거 데이터를 기반으로 분석 리포트를 생성한다. 쿼리 빈도는 낮지만, 각 쿼리가 처리하는 데이터 양은 방대하다. 컬럼 단위 읽기에 최적화되어야 한다.

이 둘 사이의 **근본적 트레이드오프**는 다음과 같다:

- **분석 리포트를 효율적으로 만들려면** 컬럼을 독립적으로 읽을 수 있어야 하므로 → 대부분의 OLAP 시스템은 **컬럼형(columnar)** 저장
- **행 단위 연산(INSERT, UPDATE)을 빠르게 하려면** 한 행의 모든 컬럼이 물리적으로 함께 저장되어야 하므로 → 대부분의 OLTP 시스템은 **행형(row-oriented)** 저장

ClickHouse는 처음부터 "가능한 한 빠른 OLAP 시스템"으로 설계되었다. 완전한 트랜잭션 지원은 없지만, consistent read/write와 mutation(UPDATE/DELETE) 기능이 점진적으로 추가되고 있다.

---

## 1.3 컬럼 지향 저장의 이점

100개의 컬럼을 가진 테이블에서 5개 컬럼만 필요한 분석 쿼리를 생각해 보자.

**행 지향 저장(Row-oriented)에서는:** 100개 컬럼의 데이터를 모두 디스크에서 읽은 다음, 필요한 5개만 추출한다. I/O의 95%가 낭비된다.

**컬럼 지향 저장(Column-oriented)에서는:** 필요한 5개 컬럼의 파일만 읽는다. I/O가 정확히 필요한 만큼만 발생한다.

이것만이 아니다. 컬럼 지향 저장에서는 같은 타입, 같은 분포의 데이터가 물리적으로 연속 배치되므로 **압축률이 극적으로 향상**된다. 예를 들어 `status` 컬럼에 "active", "inactive" 두 값만 있다면, 행 지향에서는 다른 컬럼 데이터 사이에 흩어져 있어 압축이 어렵지만, 컬럼 지향에서는 동일 패턴이 연속되어 매우 높은 압축률을 달성한다. 압축은 곧 디스크 I/O 감소이며, 이는 쿼리 성능 향상으로 직결된다.

---

## 1.4 Vectorized Query Execution: ClickHouse의 핵심 엔진

ClickHouse 성능의 핵심은 **벡터화 쿼리 실행(vectorized query execution)**이다. 개별 값(row-at-a-time)이 아닌, **배열(vector/chunk) 단위로 연산을 수행**한다.

공식 문서에서 이 아이디어의 계보를 추적한다:

> "This idea is not new. It dates back to the APL (A programming language, 1957) and its descendants: A+ (APL dialect), J (1990), K (1993), and Q (programming language from Kx Systems, 2003)."

즉, 배열 프로그래밍은 과학 데이터 처리에서 수십 년간 검증된 접근 방식이며, ClickHouse는 이를 DBMS에 전면 적용한 것이다.

### 벡터화 실행이 빠른 이유

1. **CPU 캐시 활용**: 같은 타입의 값들이 연속된 메모리에 배치되므로 L1/L2 캐시 히트율이 높다.
2. **SIMD 명령어 활용**: 하나의 CPU 명령으로 여러 값을 동시 처리한다. ClickHouse는 하드웨어에 따라 가장 최신의 SIMD 명령 세트를 자동 선택한다.
3. **함수 호출 오버헤드 감소**: 행 하나마다 함수를 호출하는 대신, 수천 개 값의 배열에 한 번만 함수를 적용한다.

### Vectorized Execution vs Runtime Code Generation

쿼리 처리 속도를 높이는 두 가지 접근이 있다:

| 접근 방식 | 원리 | 장점 | 단점 |
|-----------|------|------|------|
| **Vectorized Execution** | 데이터를 배열 단위로 처리 | SIMD 활용 용이, 구현이 상대적으로 단순 | 중간 벡터가 L2 캐시를 초과하면 성능 저하 |
| **Runtime Code Generation** | 쿼리 실행 시 기계어 코드를 동적 생성 | 여러 연산을 하나로 융합(fuse), 간접 호출 제거 | 코드 생성 오버헤드, 구현 복잡 |

공식 문서는 CMU 연구 논문을 인용하며 **두 접근의 결합이 최선**임을 제시한다. ClickHouse는 **벡터화 실행을 주축**으로 하면서, **제한적인 런타임 코드 생성을 보조적으로 사용**한다.

---

## 1.5 스토리지 레이어: INSERT와 SELECT의 격리

ClickHouse의 스토리지 아키텍처는 세 가지 핵심 성질을 갖는다:

### (1) INSERT는 즉시 새로운 "파트(Part)"를 생성한다

데이터를 INSERT하면 프라이머리 키 순서로 정렬된 새 파트가 디스크에 바로 기록된다. MEMTABLE이나 WAL(Write-Ahead Log)이 없다. 이것이 MergeTree가 LSM Tree와 다른 결정적인 차이다.

### (2) INSERT와 SELECT는 완전히 격리된다

INSERT는 새로운 파트를 생성할 뿐 기존 데이터를 건드리지 않으므로, 동시 실행되는 SELECT 쿼리에 전혀 영향을 주지 않는다. SELECT는 쿼리 시작 시점의 파트 스냅샷을 기반으로 실행된다.

### (3) 무거운 연산은 백그라운드 머지로 이관된다

파트가 계속 쌓이면 비효율적이므로, 백그라운드에서 여러 작은 파트를 하나의 큰 파트로 **머지(merge)** 한다. 핵심적인 설계 결정은, 데이터 변환 작업(중복 제거, 사전 집계, TTL 처리 등)을 INSERT 시점이 아닌 **머지 시점으로 미룬다**는 것이다. 이를 **Merge-time Computation**이라 한다.

이 접근의 이점:
- INSERT가 가볍고 빠르다 (글로벌 자료구조 갱신 불필요)
- 동시 INSERT 간 동기화 불필요
- 머지는 어차피 파트 로딩/저장이 비용의 대부분이므로, 추가 변환의 오버헤드가 미미

---

## 1.6 Data Pruning: 읽지 않는 것이 가장 빠르다

ClickHouse는 쿼리 시 불필요한 데이터를 건너뛰기 위해 세 가지 메커니즘을 제공한다:

1. **Primary Key Index (Sparse Index)**: 테이블 데이터의 정렬 순서를 정의한다. 잘 선택된 프라이머리 키는 WHERE 절 필터를 풀 스캔이 아닌 이진 탐색으로 처리하게 한다. 스캔 비용이 O(n)에서 O(log n)으로 줄어든다.

2. **Table Projections**: 동일 데이터를 다른 프라이머리 키로 정렬한 내부 복제본이다. 빈번한 필터 조건이 여러 개일 때 유용하다. 대신 데이터 중복이 발생한다.

3. **Skipping Indexes**: 컬럼에 추가 통계(최솟값/최댓값, 고유값 집합 등)를 내장한다. 프라이머리 키와 직교하는 보조 수단이다.

> "The fastest way to read data is to not read it at all."

---

## 1.7 수직 확장과 수평 확장

**수직 확장 (Vertical Scaling)**: ClickHouse는 쿼리 플랜을 여러 lane으로 펼쳐 코어당 하나씩 병렬 실행한다. 각 lane은 테이블 데이터의 서로 다른 범위를 처리하므로, CPU 코어 수에 비례해 성능이 향상된다.

**수평 확장 (Horizontal Scaling)**: 단일 노드로 부족하면 클러스터를 구성한다. 테이블을 **샤드(shard)** 단위로 분할하고 여러 노드에 분산한다. 쿼리는 데이터가 있는 모든 노드에서 실행된다.

다만 공식 문서에서 명시적으로 언급하는 제약이 있다: **클러스터는 elastic하지 않다.** 새 샤드를 추가해도 데이터가 자동으로 리밸런싱되지 않는다. 이는 수십 노드 수준에서는 괜찮지만, 수백 노드 규모에서는 significant drawback이라고 공식 문서가 인정한다.

---

## 1.8 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **정체성** | OLAP 극단에 위치한 컬럼 지향 DBMS. 완전한 트랜잭션 미지원. |
| **실행 모델** | Vectorized query execution + 제한적 runtime code generation |
| **스토리지 설계** | MergeTree ≠ LSM Tree. MEMTABLE/WAL 없음. INSERT → 즉시 디스크 파트 생성. |
| **격리 모델** | INSERT-INSERT 간, INSERT-SELECT 간 완전 격리 (immutable 파트 기반) |
| **핵심 최적화** | Merge-time computation으로 무거운 연산을 백그라운드로 이관 |
| **데이터 프루닝** | Sparse primary index, Projections, Skipping indexes — 읽지 않는 것이 최고 |
| **확장 모델** | 수직(멀티코어 병렬) + 수평(샤딩). 단, 자동 리밸런싱 없음. |

---

## 1.9 운영 시 주의점 (미리보기)

1장의 설계 철학에서 바로 도출되는 운영 원칙들:

- **배치 INSERT 필수**: 파트가 INSERT마다 생성되므로, 초당 수천 번의 소량 INSERT는 파트 폭증으로 이어진다. 배치로 모아서 넣거나 async insert를 사용하라.
- **Mutation(UPDATE/DELETE) 최소화**: MergeTree는 immutable 파트 기반이다. ALTER TABLE UPDATE/DELETE는 파트 전체를 다시 쓰는 무거운 연산이다.
- **프라이머리 키 설계가 곧 성능**: 테이블 생성 후 ORDER BY 키를 변경할 수 없다. 쿼리 패턴을 사전에 파악하고 설계해야 한다.
- **클러스터 리밸런싱 부재 인지**: 샤드 추가 시 수동으로 데이터를 재배치해야 한다. 초기 샤딩 전략이 중요하다.

---

*다음 장: 2장. 내부 아키텍처 컴포넌트 — Column, Block, Parser, Interpreter, Processor의 세계*

---

## 1.10 시각적 이해 — 행 지향 vs 컬럼 지향

실제 메모리/디스크 레이아웃을 비교하면 컬럼 지향이 왜 OLAP에 유리한지 명확해진다.

### 예시 테이블

```
user_id | name    | age | country | amount
--------+---------+-----+---------+--------
  1001  | Alice   |  25 | Korea   | 50000
  1002  | Bob     |  31 | USA     | 12000
  1003  | Charlie |  42 | Japan   | 78000
  1004  | Dave    |  29 | Korea   | 35000
```

### 행 지향 저장 (MySQL, PostgreSQL)

```
디스크 상 실제 배치:

[1001][Alice][25][Korea][50000] | [1002][Bob][31][USA][12000] | [1003][Charlie][42][Japan][78000] | [1004][Dave][29][Korea][35000]
 └─ 한 행의 모든 컬럼이 연속 배치 ─┘

쿼리: SELECT sum(amount) WHERE country='Korea'
  → 모든 행의 모든 컬럼을 읽어야 함
  → 필요 없는 name, age까지 전부 스캔
  → I/O 낭비 심각
```

### 컬럼 지향 저장 (ClickHouse)

```
디스크 상 실제 배치 (컬럼별 파일 분리):

user_id.bin:  [1001][1002][1003][1004]
name.bin:     [Alice][Bob][Charlie][Dave]
age.bin:      [25][31][42][29]
country.bin:  [Korea][USA][Japan][Korea]
amount.bin:   [50000][12000][78000][35000]

쿼리: SELECT sum(amount) WHERE country='Korea'
  → country.bin, amount.bin만 읽음
  → name.bin, age.bin은 디스크 터치 안 함
  → I/O 최소화
```

### 압축률 차이

```
country 컬럼만 봤을 때:

행 지향 (컬럼 섞임): 
  Korea, USA, Japan, Korea, ...
  → 주변 데이터와 섞여 있어 압축 패턴 찾기 어려움
  → LZ4 압축률 ~50%

컬럼 지향 (같은 컬럼 연속):
  Korea, Korea, Korea, Korea, USA, USA, Japan, Japan, ...
  → 반복 패턴 명확 → 압축 효과 극대화
  → LowCardinality + LZ4 압축률 ~95%
```

---

## 1.11 시각적 이해 — MergeTree vs LSM Tree

ClickHouse가 LSM Tree가 아니라는 점은 반복 강조됐다. 구조적 차이를 시각화.

### LSM Tree (RocksDB, Cassandra)

```
INSERT
   │
   ▼
┌─────────────────┐
│   MEMTABLE      │  ← 인메모리 쓰기 버퍼
│  (정렬된 맵)     │
└────────┬────────┘
         │ 동시에 WAL(Write-Ahead Log)에 기록 → 장애 복구용
         │
         │ MEMTABLE이 가득 차면...
         ▼
┌─────────────────┐
│  Level 0 SSTable│  ← 디스크로 flush
└────────┬────────┘
         │ Compaction
         ▼
┌─────────────────┐
│  Level 1 SSTable│
└─────────────────┘

장점: 소량 빈번 INSERT OK (MEMTABLE에 누적)
단점: WAL 오버헤드, MEMTABLE 크기 관리
```

### MergeTree (ClickHouse)

```
INSERT (배치 단위)
   │
   │ MEMTABLE 없음. WAL 없음.
   ▼
┌─────────────────┐
│  Part (level 0) │  ← 즉시 디스크에 기록
│  (정렬된 파트)  │
└────────┬────────┘
         │ 백그라운드 머지
         ▼
┌─────────────────┐
│  Part (level 1) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Part (level N) │
└─────────────────┘

장점: 단순, WAL 오버헤드 없음, 배치 INSERT 초고속
단점: 소량 빈번 INSERT 금물 → Async Insert 필요
```

### 핵심 차이

```
LSM Tree:
  INSERT → 메모리 → (시간 후) 디스크
  읽기 시 MEMTABLE + 여러 SSTable 레벨 확인

MergeTree:
  INSERT → 즉시 디스크 (정렬된 파트)
  읽기 시 여러 파트 병렬 스캔
  
"왜 다른가?"
  LSM: OLTP 스타일 (빈번한 소량 쓰기 최적화)
  MergeTree: OLAP 스타일 (배치 쓰기, 대용량 읽기 최적화)
```
