# ClickHouse 아키텍처 & 운영 학습 가이드

> 공식 문서(architecture.md, core-concepts, best-practices) 기반 정리
> Key Takeaway · Architecture · 컴포넌트 · 운영 시 주의점 중심

---

## 목차

### 1부: 아키텍처 기초
- **1장.** ClickHouse 개요와 설계 철학
- **2장.** 내부 아키텍처 컴포넌트 — Column, Block, Parser, Interpreter, Processor
- **3장.** Context, Thread Pool, 동시성 제어

### 2부: 스토리지 엔진 — MergeTree 깊이 파기
- **4장.** Table Parts — MergeTree 스토리지의 물리적 구조
- **5장.** Table Partitions — 데이터 Lifecycle 관리의 단위
- **6장.** Table Part Merges — 백그라운드 머지의 동작 원리
- **7장.** Primary Index — Sparse Index의 원리와 ORDER BY 키 설계

### 3부: 분산 아키텍처
- **8장.** Shards와 Replicas — 분산 아키텍처

### 4부: 운영 실무
- **9장.** INSERT 전략 선택
- **10장.** 피해야 할 패턴들
- **11장.** 모니터링과 트러블슈팅
- **부록.** 참고 자료

---

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
# 2장. 내부 아키텍처 컴포넌트

이 장에서는 ClickHouse 서버 내부의 핵심 컴포넌트들을 하나씩 살펴본다. SQL 쿼리가 입력되어 결과가 반환되기까지의 흐름을 따라가면서, 각 단계에서 어떤 추상화가 사용되는지 이해하는 것이 목표다.

전체 흐름을 먼저 조감하면 다음과 같다:

```
SQL 문자열
  → [Parser] → AST (추상 구문 트리)
    → [Interpreter] → QueryPipeline (Processor들의 그래프)
      → [PipelineExecutor] → Block 단위 데이터 처리
        → [IStorage] → 디스크 I/O (MergeTree 파트 읽기/쓰기)
          → 결과 출력 (Format → WriteBuffer → 클라이언트)
```

---

## 2.1 데이터 표현: Column, Field, DataType, Block

쿼리 실행의 가장 기본이 되는 4가지 자료 구조를 이해해야 한다.

### IColumn — 메모리 상의 컬럼

`IColumn`은 메모리에서 컬럼(정확히는 컬럼의 청크)을 표현하는 인터페이스다. 핵심 특성은 **거의 모든 연산이 immutable**이라는 점이다. 원본 컬럼을 수정하지 않고 새로운 컬럼을 생성한다.

```
IColumn::filter(mask)   → WHERE / HAVING 구현
IColumn::permute(perm)  → ORDER BY 구현
IColumn::cut(start, n)  → LIMIT 구현
```

구체적인 구현 클래스별 메모리 레이아웃:

| 구현 클래스 | 메모리 레이아웃 |
|------------|----------------|
| `ColumnUInt8`, `ColumnUInt64` 등 | 하나의 연속 배열 (`std::vector`와 동일) |
| `ColumnString`, `ColumnArray` | 두 개의 벡터: ① 모든 원소가 연속 배치된 데이터 벡터 ② 각 원소의 시작 오프셋 벡터 |
| `ColumnConst` | 메모리에 값 하나만 저장하되, 컬럼처럼 동작 |

이 레이아웃이 중요한 이유는, 정수형 컬럼의 경우 데이터가 물리적으로 연속된 메모리에 놓이므로 SIMD 연산을 직접 적용할 수 있기 때문이다.

### Field — 개별 값 표현 (비효율적)

개별 값을 다루어야 할 때 사용하는 `Field`는 `UInt64 | Int64 | Float64 | String | Array`의 **discriminated union**이다. `IColumn`의 `operator[]`로 n번째 값을 `Field`로 꺼낼 수 있지만, 공식 문서는 이를 "not very efficient"라고 명시한다.

주의할 점: `Field`는 구체적인 데이터 타입 정보를 갖지 않는다. `UInt8`, `UInt16`, `UInt32`, `UInt64` 모두 `Field` 내에서는 `UInt64`로 표현된다. 타입 정보가 필요하면 `IDataType`을 참조해야 한다.

### Leaky Abstractions — 성능을 위한 의도적 추상화 누수

`IColumn` 인터페이스만으로는 모든 연산을 효율적으로 구현할 수 없다. 예를 들어 `ColumnUInt64`에 "두 컬럼의 합산" 메서드는 없고, `ColumnString`에 "부분 문자열 검색" 메서드도 없다.

두 가지 구현 방식이 있다:

1. **제네릭 방식**: `IColumn` 메서드로 `Field` 값을 추출하여 연산 → 느림
2. **특화 방식**: 특정 `IColumn` 구현의 내부 메모리 레이아웃을 직접 접근 → 빠름

ClickHouse는 성능을 위해 2번을 적극 사용한다. `ColumnUInt64::getData()`가 내부 배열의 참조를 반환하면, 외부 루틴이 그 배열을 직접 읽고 쓴다. 공식 문서는 이를 **"leaky abstractions to allow efficient specializations"**이라 표현한다.

이것은 설계 결함이 아니라 의도적 선택이다. 추상화의 순수성보다 실행 성능을 우선하는 ClickHouse의 철학을 보여준다.

### IDataType — 직렬화/역직렬화 담당

`IDataType`은 테이블의 데이터 타입에 직접 대응된다 (`DataTypeUInt32`, `DataTypeDateTime`, `DataTypeString` 등). 핵심 역할은 **컬럼 청크나 개별 값의 바이너리/텍스트 형식 읽기·쓰기**이다.

중요한 설계 결정: **`IDataType`과 `IColumn`은 느슨하게 결합**되어 있다.

```
DataTypeUInt32  ─┐
                 ├── ColumnUInt32 (또는 ColumnConstUInt32)
DataTypeDateTime ─┘

DataTypeUInt8 ──── ColumnUInt8 또는 ColumnConstUInt8
```

같은 `IColumn` 구현이 여러 `IDataType`에 사용될 수 있고, 같은 `IDataType`이 여러 `IColumn` 구현으로 표현될 수 있다. `IDataType` 자체는 메타데이터만 저장한다. `DataTypeUInt8`은 vptr 외에 아무것도 저장하지 않고, `DataTypeFixedString`은 고정 크기 `N`만 저장한다.

### Block — 쿼리 실행의 기본 단위

`Block`은 테이블의 부분 집합(청크)을 메모리에 표현하는 컨테이너다. 구조는 단순하다:

```
Block = { (IColumn, IDataType, column_name), (IColumn, IDataType, column_name), ... }
```

쿼리 실행 중 데이터는 `Block` 단위로 처리된다. 함수를 실행하면 결과 컬럼이 Block에 **추가**되고, 인자 컬럼은 건드리지 않는다 (immutable). 나중에 불필요한 컬럼을 Block에서 제거할 수는 있지만 수정은 없다. 이 설계는 **공통 부분식 제거(common subexpression elimination)** 최적화에 유리하다.

같은 유형의 연산에서는 Block 간에 컬럼 이름과 타입이 동일하고, 컬럼 데이터만 바뀐다. 따라서 Block 헤더(이름, 타입)와 데이터를 분리하면 오버헤드를 줄일 수 있다.

---

## 2.2 쿼리 파싱: Parser → AST

ClickHouse는 **hand-written recursive descent parser**를 사용한다. Parser generator(yacc, ANTLR 등)는 "역사적 이유로" 사용하지 않는다.

예를 들어 `ParserSelectQuery`는 SELECT 쿼리의 각 부분(컬럼 목록, FROM, WHERE, GROUP BY 등)에 대한 하위 파서를 재귀적으로 호출한다. 파싱 결과는 **AST(Abstract Syntax Tree)**로, 각 노드는 `IAST` 인터페이스의 인스턴스다.

Hand-written parser의 장점은 에러 메시지를 세밀하게 제어할 수 있고, 특수 구문을 유연하게 추가할 수 있다는 점이다. ClickHouse가 수백 가지의 함수와 독자적 SQL 확장을 지원하는 데 이 유연성이 기여한다.

---

## 2.3 쿼리 해석: Interpreter → QueryPipeline

**Interpreter**는 AST를 받아 쿼리 실행 파이프라인을 생성한다. 단순한 것(`InterpreterExistsQuery`, `InterpreterDropQuery`)부터 복잡한 것(`InterpreterSelectQuery`)까지 다양하다.

### QueryPipeline의 세 가지 모드

| 모드 | 설명 | 사용 예 |
|------|------|---------|
| **Pulling** | 특수 출력 포트에서 결과를 "당겨" 읽음 | `SELECT` 쿼리 |
| **Pushing** | 입력 포트에 데이터를 "밀어" 넣음 | `INSERT` 쿼리 |
| **Completed** | 입력도 출력도 없음, 내부에서 자체 실행 | `INSERT SELECT` (SELECT → INSERT 데이터 복사) |

### 쿼리 최적화: ExpressionAnalyzer → QueryTree

`InterpreterSelectQuery`는 `ExpressionAnalyzer`와 `ExpressionActions`를 사용해 쿼리 분석과 변환을 수행한다. **규칙 기반 쿼리 최적화**가 이 단계에서 일어난다.

공식 문서는 `ExpressionAnalyzer`에 대해 솔직하게 평가한다:

> "ExpressionAnalyzer is quite messy and should be rewritten: various query transformations and optimizations should be extracted into separate classes to allow for modular transformations of the query."

이 문제를 해결하기 위해 새로운 `InterpreterSelectQueryAnalyzer`가 개발되었다. 핵심 변화는 AST와 QueryPipeline 사이에 **`QueryTree`**라는 추가 추상화 레이어를 도입한 것이다. 이미 프로덕션에서 사용할 준비가 되어 있으며, `enable_analyzer` 설정으로 제어한다(기본값 true).

```
[기존] AST → ExpressionAnalyzer → QueryPipeline
[신규] AST → QueryTree → InterpreterSelectQueryAnalyzer → QueryPipeline
```

---

## 2.4 쿼리 실행: Processor와 Pipeline

### Processor 모델

Processor는 **포트(port)**를 통해 통신하며, 여러 개의 입력 포트와 출력 포트를 가질 수 있다. 각 Processor는 Block(컬럼 청크)을 소비하고 생산한다.

```
[Source Processor] → [Filter Processor] → [Aggregation Processor] → [Output Processor]
     (읽기)              (WHERE)              (GROUP BY)              (결과 전송)
```

여러 Processor를 연결하면 DAG(방향 비순환 그래프) 형태의 `QueryPipeline`이 만들어지고, `PipelineExecutor`가 이를 실행한다.

### 병렬 실행

테이블의 `read` 메서드가 여러 `Processor`를 반환하면, 이들은 테이블에서 **병렬로** 데이터를 읽는다. 각 Processor에 독립적으로 필터링, 표현식 평가 등의 변환을 연결할 수 있다. 이것이 ClickHouse가 멀티코어를 활용하는 메커니즘이다.

---

## 2.5 Functions: Ordinary와 Aggregate

### Ordinary Functions

일반 함수는 **행 수를 변경하지 않는다**. 개념적으로는 행마다 독립적으로 작동하지만, 실제로는 `Block` 단위의 벡터화 실행이다.

주의해야 할 특성:

**강타입(Strong Typing)**: 암묵적 타입 변환이 없다. 지원하지 않는 타입 조합이면 예외를 던진다. 대신 `plus` 함수처럼 수많은 타입 조합에 대한 오버로드를 제공한다 (`UInt8 + Float32`, `UInt16 + Int8` 등).

**Short-circuit 평가 없음**: `WHERE f(x) AND g(y)`에서 `f(x)`가 false여도 `g(y)`가 계산된다 (벡터화 실행의 특성). `f(x)`의 선택성이 높고 `g(y)`가 비용이 크다면, 멀티패스 계산(`f(x)` 먼저 → 필터링 → 필터된 데이터에만 `g(y)`)이 더 효율적이다.

### Aggregate Functions

집계 함수는 **상태를 가지는(stateful)** 함수다. `IAggregateFunction` 인터페이스로 관리된다.

상태의 복잡도는 다양하다:
- `AggregateFunctionCount`: `UInt64` 하나
- `AggregateFunctionUniqCombined`: 선형 배열 + 해시 테이블 + HyperLogLog의 조합

고카디널리티 `GROUP BY` 실행 시 수많은 상태를 관리해야 하므로, **`Arena`(메모리 풀)**에서 상태를 할당한다.

분산 쿼리 실행 시 집계 상태를 **직렬화/역직렬화**해서 네트워크를 통해 전송하거나, RAM이 부족할 때 디스크에 쓸 수 있다. `DataTypeAggregateFunction`을 사용하면 테이블에 집계 상태를 직접 저장할 수도 있는데, 이것이 `AggregatingMergeTree`의 증분 집계 기능의 기반이다.

공식 문서의 주의사항:

> "The serialized data format for aggregate function states is not versioned right now. ... we have the AggregatingMergeTree table engine for incremental aggregation, and people are already using it in production. It is the reason why backward compatibility is required when changing the serialized format for any aggregate function in the future."

→ `AggregatingMergeTree`를 사용 중이라면, ClickHouse 버전 업그레이드 시 집계 함수의 직렬화 포맷 호환성을 확인해야 한다.

---

## 2.6 Table Engine: IStorage 인터페이스

`IStorage` 인터페이스가 테이블을 표현한다. 구현체가 곧 테이블 엔진이다 (`StorageMergeTree`, `StorageMemory`, `StorageDistributed` 등).

핵심 메서드:

```
IStorage::read(columns, ast_query, num_streams) → Pipe
IStorage::write(...)
IStorage::alter(...), rename(...), drop(...)
```

`read` 메서드의 중요한 설계 원칙: **대부분의 경우 테이블에서 지정된 컬럼을 읽는 것만 담당**하고, 그 이후의 데이터 처리는 파이프라인의 다른 부분이 담당한다.

그러나 두 가지 주목할 예외가 있다:

1. **인덱스 활용**: AST 쿼리가 `read`에 전달되므로, 테이블 엔진이 인덱스를 판단해 더 적은 데이터를 읽을 수 있다.
2. **분산 쿼리 푸시다운**: `StorageDistributed`는 쿼리를 원격 서버로 보내 특정 단계까지 처리한 결과를 받아온다. Interpreter가 나머지를 마무리한다.

`read`가 반환하는 `Pipe`는 여러 `Processor`로 구성될 수 있고, 이들은 테이블에서 병렬로 읽는다. `IStorage`는 `QueryProcessingStage`도 반환하여, 쿼리의 어느 부분까지 스토리지 내에서 이미 처리되었는지 알려준다.

---

## 2.7 I/O와 Format

### ReadBuffer / WriteBuffer

바이트 수준 I/O에 C++ `iostream` 대신 자체 `ReadBuffer`/`WriteBuffer`를 사용한다. 구조는 **연속 버퍼 + 커서**로 단순하며, 구현체에 따라 파일, 소켓, 압축 등을 처리한다.

```
CompressedWriteBuffer → 다른 WriteBuffer를 감싸서 압축 후 기록
ConcatReadBuffer      → 여러 ReadBuffer를 연결
LimitReadBuffer       → 읽기 크기 제한
HashingWriteBuffer    → 쓰면서 해시 계산
```

### 결과 출력 흐름 예시: JSON → stdout

```
QueryPipeline (pulling)
  → JSONRowOutputFormat (WriteBuffer를 감싸 JSON 형식으로 행 출력)
    → WriteBufferFromFileDescriptor(STDOUT_FILENO)
      → stdout
```

내부적으로 `JSONRowOutputFormat`은 `IDataType::serializeTextJSON`을 호출하고, 이것이 `WriteHelpers.h`의 `writeText`(숫자)나 `writeJSONString`(문자열)을 호출한다.

### Format Processor

데이터 포맷(JSON, CSV, Parquet 등)은 Processor로 구현된다. 즉, 포맷 변환도 파이프라인의 일부로 통합되어 있다.

---

## 2.8 Server 인터페이스

ClickHouse 서버는 세 가지 네트워크 인터페이스를 제공한다:

| 인터페이스 | 용도 | 특징 |
|-----------|------|------|
| **HTTP** | 외부 클라이언트 (대부분의 애플리케이션) | 단순하고 사용 쉬움. **공식 권장** |
| **TCP (Native)** | ClickHouse 네이티브 클라이언트, 서버 간 분산 쿼리 통신 | 내부 Block 포맷 사용, 커스텀 압축 프레이밍 |
| **Replication** | 레플리카 간 데이터 전송 | 압축된 파트 단위 전송 |

서버 자체는 코루틴이나 파이버 없는 **단순한 멀티스레드 서버**다. 이는 의도적 설계다: ClickHouse는 대량의 단순 쿼리가 아닌, 소수의 복잡한 쿼리(각각 방대한 데이터 처리)를 위해 설계되었기 때문이다.

TCP 프로토콜은 **완전한 전·후방 호환성**을 유지한다 (구 클라이언트 ↔ 신 서버, 신 클라이언트 ↔ 구 서버). 단, 영원히는 아니고 약 1년 후 구 버전 지원을 제거한다.

---

## 2.9 Configuration 시스템

서버 설정은 POCO C++ 라이브러리의 `Poco::Util::AbstractConfiguration`으로 표현된다.

핵심 동작:

- **다중 파일 병합**: XML 또는 YAML 형식의 여러 설정 파일을 `ConfigProcessor`가 하나의 `AbstractConfiguration`으로 병합
- **런타임 리로드**: `ConfigReloader`가 주기적으로 파일 변경을 감시. `SYSTEM RELOAD CONFIG` 쿼리로도 리로드 가능
- **에러 내성**: 새 설정에 에러가 있으면 대부분의 서브시스템이 경고 로그를 남기고 **이전 설정을 유지**

### Context 계층 구조

ClickHouse의 설정은 4단계 계층으로 관리되며, 아래 단계가 위를 오버라이드한다:

```
① Global defaults (코드 내 기본값)
  ② Global configuration (설정 파일)
    ③ Profile settings (<profiles> 섹션)
      ④ User settings (<users> 섹션)
        ⑤ Session settings (SET 커맨드)
          ⑥ Query settings (SETTINGS 절)
```

서버가 쿼리를 실행할 때 이 순서로 설정을 머지하여 최종 Context를 생성한다.

**Background context 주의**: 머지, 뮤테이션 등 백그라운드 작업은 Global + 'background' 프로파일 설정만 적용된다. Session/Query 설정은 영향을 주지 않는다. 기본 프로파일명은 `'background'`이며, `background_profile` 서버 설정으로 변경 가능하다.

---

## 2.10 Key Takeaways

| 컴포넌트 | 핵심 포인트 |
|----------|------------|
| **IColumn** | 벡터화 실행의 기반. Immutable 연산. 내부 메모리 직접 접근(leaky abstractions)으로 성능 확보 |
| **Block** | (IColumn, IDataType, name) 트리플. 쿼리 실행의 데이터 전달 단위 |
| **Parser** | Hand-written recursive descent. AST 생성 |
| **Interpreter** | AST → QueryPipeline. 새로운 QueryTree 추상 레이어 도입 중 (`enable_analyzer`) |
| **Processor** | 멀티포트 DAG 기반 실행 모델. 병렬 읽기와 변환 |
| **Functions** | 강타입, short-circuit 없음. 집계 함수는 Arena에서 상태 관리 |
| **IStorage** | 테이블 엔진 인터페이스. 읽기만 담당하고 나머지는 파이프라인에 위임 (예외: 인덱스, 분산 푸시다운) |
| **Server** | HTTP(권장) / TCP(내부) / Replication. 단순 멀티스레드, 복잡한 소수 쿼리에 최적화 |
| **Config** | 다중 파일 → 병합 → 런타임 리로드. 에러 시 이전 설정 유지 |
| **Context** | 6단계 설정 머지. Background 작업은 Session/Query 설정 무시 |

---

## 2.11 운영 시 주의점

- **`enable_analyzer` 확인**: 새 분석기(QueryTree 기반)는 기본 활성화되어 있다. 기존 쿼리와의 동작 차이가 발생하면 `enable_analyzer = false`로 전환 가능하나, 장기적으로는 새 분석기가 표준이 될 것이다.
- **Background 프로파일 설정**: 머지/뮤테이션의 동작을 튜닝하려면 Session SET이 아닌, `<profiles>` 섹션의 `background` 프로파일을 수정해야 한다. 쿼리 레벨 SETTINGS로는 제어되지 않는다.
- **설정 리로드 에러 내성**: 설정 파일 문법 오류가 있어도 서버가 죽지 않고 이전 설정으로 동작한다. 이는 안전하지만, 설정 변경이 적용되지 않은 채 운영되는 상황을 간과하기 쉽다. 설정 변경 후 `system.settings` 테이블로 실제 적용 여부를 확인하라.
- **HTTP vs TCP 선택**: 외부 애플리케이션은 HTTP를 사용하라. TCP는 내부 Block 포맷에 강하게 결합되어 있어 C 라이브러리조차 제공하지 않는다.
- **Aggregate Function 직렬화 호환성**: `AggregatingMergeTree`로 증분 집계를 사용 중이면, ClickHouse 메이저 버전 업그레이드 시 집계 상태의 직렬화 포맷 호환성을 반드시 확인하라.

---

*다음 장: 3장. Context, Thread Pool, 동시성 제어 — 서버의 리소스 관리 체계*
# 3장. Context, Thread Pool, 동시성 제어

ClickHouse 서버는 단일 프로세스 안에서 수많은 작업을 동시에 처리한다. 사용자 쿼리 실행, 백그라운드 머지, 뮤테이션, 레플리카 간 데이터 페치, S3 백업 등이 모두 하나의 서버 프로세스 내에서 벌어진다. 이 장에서는 이 작업들을 조율하는 세 가지 핵심 메커니즘을 다룬다: Context(설정 계층), Thread Pool(스레드 관리), ConcurrencyControl(CPU 자원 경쟁 해소).

---

## 3.1 Context 계층 구조: 설정은 어디서 오는가

2장에서 간략히 언급한 Context를 여기서 깊이 다룬다. ClickHouse의 모든 동작은 설정(settings)에 의해 제어되며, 설정의 출처는 여러 단계에 걸쳐 있다.

### 4가지 Context 유형

| Context | 범위 | 설정 출처 |
|---------|------|-----------|
| **Global context** | 서버 전체 | config.xml / config.yaml 등 설정 파일 |
| **Session context** | 사용자 세션 | profiles, users 설정 + `SET` 커맨드 |
| **Query context** | 개별 쿼리 | `SETTINGS` 절 |
| **Background context** | 백그라운드 작업 | Global + 'background' 프로파일 |

### 설정 머지 순서

서버가 쿼리를 실행할 때 설정을 아래 순서로 머지한다. **뒤의 것이 앞을 오버라이드**한다:

```
① Global defaults (코드 내 하드코딩된 기본값)
  ② Global configuration (설정 파일)
    ③ Profile settings (<profiles> 섹션)
      ④ User settings (<users> 섹션)
        ⑤ Session settings (SET 커맨드)
          ⑥ Query settings (SETTINGS 절)
```

이 순서를 이해하면, "왜 SET으로 바꾼 설정이 특정 쿼리에서 안 먹히는지", "왜 config 파일을 바꿨는데 동작이 안 바뀌는지" 등의 혼란을 피할 수 있다.

### Background Context의 특수성

이것은 운영에서 자주 혼동되는 부분이다. 머지, 뮤테이션 같은 백그라운드 작업은 **Session context와 Query context를 무시**한다. 오직 Global context와 'background' 프로파일만 적용된다.

공식 문서 원문:

> "Background operations can be configured via global and 'background' profile settings; session and query settings have no effect in this case."

따라서 다음과 같은 시나리오가 발생한다:

**잘못된 시도**: `SET max_bytes_to_merge_at_max_space_in_pool = ...` → 세션 설정이므로 백그라운드 머지에 영향 없음

**올바른 방법**: `<profiles>` 섹션의 `background` 프로파일에 해당 설정 추가, 또는 `background_profile` 서버 설정으로 다른 프로파일 지정

---

## 3.2 Thread Pool 계층 구조

ClickHouse는 스레드 생성·소멸 비용을 피하기 위해 **스레드 풀(thread pool)** 체계를 사용한다. 모든 특수 풀은 Global thread pool 위에 구축된 계층 구조를 이룬다.

```
Global Thread Pool (최상위, 싱글톤)
  ├── Server Pool (클라이언트 커넥션)
  ├── IO Thread Pool (I/O 집중 작업)
  │     └── Backups IO Thread Pool (S3 백업 전용)
  ├── Background Schedule Pool (주기적 작업)
  └── MergeTreeBackgroundExecutor (머지/뮤테이션/페치/무브)
```

### Global Thread Pool

모든 풀의 기반이 되는 `GlobalThreadPool` 싱글톤이다. `ThreadFromGlobalPool`로 스레드를 할당받으며, `std::thread`와 유사한 인터페이스를 제공한다.

설정:
- `max_thread_pool_size`: 풀 내 최대 스레드 수
- `max_thread_pool_free_size`: 새 작업을 대기하는 유휴 스레드 수 상한
- `thread_pool_queue_size`: 대기열에 쌓을 수 있는 작업 수 상한

모든 하위 특수 풀은 Global pool에서 스레드를 가져온다. 특수 풀의 역할은 **동시 작업 수를 제한**하고 **작업 스케줄링**을 하는 것이다. 작업이 스레드보다 많으면 우선순위 큐에 쌓이며, 우선순위 값이 높은 작업이 먼저 시작된다 (기본 우선순위는 0). 단, 이미 실행 중인 작업 간에는 우선순위 차이가 없다. 우선순위는 풀이 과부하 상태일 때만 의미가 있다.

### Server Pool

`Poco::ThreadPool` 기반이며, 최대 `max_connection` 개의 스레드를 가진다. **각 스레드가 하나의 활성 커넥션을 전담**한다.

### IO Thread Pool

I/O에 블로킹되는 작업(디스크 읽기/쓰기, 네트워크 대기 등)을 위한 풀이다. 존재 이유가 명확하다:

> "The main purpose of IO thread pool is to avoid exhaustion of the global pool with IO jobs, which could prevent queries from fully utilizing CPU."

I/O 작업이 Global pool의 스레드를 모두 점유하면 CPU 집중 쿼리가 스레드를 할당받지 못하는 상황이 발생한다. IO thread pool은 이를 격리한다.

설정: `max_io_thread_pool_size`, `max_io_thread_pool_free_size`, `io_thread_pool_queue_size`

S3 백업은 I/O가 특히 많으므로, 별도의 `BackupsIOThreadPool`이 존재한다 (`max_backups_io_thread_pool_size` 등). 인터랙티브 쿼리에 대한 백업의 영향을 격리하기 위함이다.

### Background Schedule Pool

`BackgroundSchedulePool`은 주기적 작업 실행을 담당한다. 핵심 보장:

- **동일 작업의 동시 실행 방지**: 한 작업이 실행 중이면 같은 작업이 다시 시작되지 않음
- **실행 시점 제어**: 미래의 특정 시점으로 실행을 미루거나 일시 비활성화 가능

`Context::getSchedulePool()`로 범용 인스턴스에 접근한다.

### MergeTreeBackgroundExecutor

MergeTree 관련 백그라운드 작업(머지, 뮤테이션, 페치, 무브)을 위한 특수 풀이다. 이 풀의 핵심 설계는 **선점 가능한 작업(preemptable tasks)** 지원이다.

`IExecutableTask`를 순서가 있는 **스텝(step)** 시퀀스로 분할할 수 있다. 이 구조의 이점은 짧은 작업이 긴 작업보다 우선 처리되도록 스케줄링할 수 있다는 것이다. 예를 들어 작은 파트의 머지가 큰 파트의 머지보다 먼저 완료되도록 할 수 있다.

`Context::getCommonExecutor()` 등으로 접근한다.

---

## 3.3 ThreadStatus와 Thread Group: 쿼리별 리소스 추적

어떤 풀에서 스레드가 할당되든, 작업 시작 시 `ThreadStatus` 인스턴스가 생성된다. 이 객체는 해당 스레드의 모든 정보를 캡슐화한다:

- Thread ID, Query ID
- Performance counters
- 리소스 소비량

Thread-local 포인터로 관리되어 `CurrentThread::get()` 호출로 어디서든 접근 가능하다. 모든 함수에 매개변수로 전달할 필요가 없다.

### 쿼리 실행과 Thread Group

쿼리 실행 시의 스레드 관계는 다음과 같다:

```
Master Thread (Server pool에서 할당)
  ├── ThreadStatus::QueryScope(query_context) 보유
  ├── ThreadGroupStatus 생성
  │
  ├── Worker Thread 1 → CurrentThread::attachTo(thread_group)
  ├── Worker Thread 2 → CurrentThread::attachTo(thread_group)
  └── Worker Thread N → CurrentThread::attachTo(thread_group)
```

- **Master thread**: 서버 풀의 스레드로, `QueryScope` 객체를 보유하여 쿼리 컨텍스트를 연결
- **Thread group**: Master thread가 생성. 쿼리 실행 중 추가 할당된 모든 스레드가 여기에 연결
- **목적**: `ProfileEvents::Counters`로 프로파일 이벤트를 집계하고, `MemoryTracker`로 쿼리 단위 메모리 소비를 추적

이 구조 덕분에 `system.query_log`에서 쿼리별 메모리 사용량, 읽기/쓰기 바이트, CPU 시간 등을 정확히 확인할 수 있다.

---

## 3.4 ConcurrencyControl: CPU Slot 기반 동시성 제어

### 문제: 동시 쿼리의 CPU 경쟁

`max_threads` 설정의 기본값은 단일 쿼리가 모든 CPU 코어를 활용하도록 설계되어 있다. 그런데 여러 쿼리가 동시에 실행되면?

각 쿼리가 `max_threads`만큼 스레드를 요청하면 OS가 스레드 간 컨텍스트 스위칭으로 공정성을 보장하지만, 이는 **성능 페널티**를 발생시킨다. 스레드가 과도하게 많아지면 캐시 미스, 컨텍스트 스위칭 비용, 메모리 오버헤드가 모두 증가한다.

### 해결: CPU Slot 모델

`ConcurrencyControl`은 **CPU slot** 개념을 도입한다. Slot은 동시성의 단위다:

- 스레드를 실행하려면 먼저 slot을 **획득(acquire)**해야 한다
- 스레드가 중단되면 slot을 **반환(release)**한다
- 서버 전체에서 slot 총 수가 **전역적으로 제한**된다

### Slot 상태 머신

각 slot은 세 가지 상태를 가진 독립적 상태 머신이다:

```
   free ──allocate──→ granted ──acquire──→ acquired
    ↑                                        │
    └──────────── release ←──────────────────┘
```

| 상태 | 설명 |
|------|------|
| `free` | 어떤 쿼리든 할당 가능 |
| `granted` | 특정 쿼리에 할당되었지만, 아직 스레드가 점유하지 않음 (짧은 과도 상태) |
| `acquired` | 특정 쿼리에 할당되고, 실제 스레드가 점유 중 |

### API

ConcurrencyControl의 사용 패턴을 코드로 보면 직관적이다:

**1단계: 리소스 할당**
```cpp
auto slots = ConcurrencyControl::instance().allocate(1, max_threads);
```
최소 1개, 최대 `max_threads`개의 slot을 요청한다. **첫 번째 slot은 즉시 부여**되지만, 나머지는 나중에 부여될 수 있다. 이것이 "soft limit"의 의미다 — 모든 쿼리는 CPU 부하가 높더라도 **최소 1개 스레드를 보장**받는다.

**2단계: 스레드별 slot 획득**
```cpp
while (auto slot = slots->tryAcquire())
    spawnThread([slot = std::move(slot)] { ... });
```
slot이 부여(granted)될 때마다 스레드를 생성한다. 초기에 1개로 시작하더라도 나중에 slot이 추가로 부여되면 스레드가 늘어난다.

**3단계: 전체 slot 수 조정 (런타임)**
```cpp
ConcurrencyControl::setMaxConcurrency(concurrent_threads_soft_limit_num);
```
서버 재시작 없이 런타임에 변경 가능하다.

### 동작 시나리오

64코어 서버에서 `concurrent_threads_soft_limit_num = 64`로 설정한 경우:

**쿼리 1개만 실행 중**: 64개 slot을 모두 획득 가능 → 모든 코어 활용

**4개 쿼리 동시 실행**: 각각 처음에 1개 slot 확보(즉시) → ConcurrencyControl이 공정하게 나머지 60개 slot을 분배 → 각 쿼리 약 16개 스레드로 수렴

**100개 쿼리 동시 실행**: 각각 최소 1개 스레드 보장 (soft limit) → 총 100개 스레드 사용. `concurrent_threads_soft_limit_num`(64)을 초과하지만, 최소 보장 때문에 허용됨. 각 쿼리는 1개 스레드로 시작하고, 다른 쿼리가 끝나면 slot이 해제되어 남은 쿼리가 스케일업

---

## 3.5 전체 그림: 쿼리 하나의 라이프사이클

지금까지의 내용을 종합하여 SELECT 쿼리 하나의 전체 라이프사이클을 추적하면:

```
1. 클라이언트 커넥션 수락
   └── Server pool에서 Master thread 할당

2. Context 구성
   └── Global → Profile → User → Session → Query 순서로 설정 머지

3. 쿼리 파싱 + 해석
   └── Parser → AST → Interpreter → QueryPipeline 생성

4. ConcurrencyControl에서 slot 할당
   └── allocate(1, max_threads) → 최소 1개 slot 즉시 확보

5. 스레드 할당 + 실행
   └── Master thread가 ThreadGroupStatus 생성
   └── 획득된 slot 수만큼 Worker thread 생성 (Global pool에서)
   └── 각 Worker가 thread group에 attach → MemoryTracker, ProfileEvents 집계

6. Pipeline 실행
   └── PipelineExecutor가 Processor DAG 실행
   └── IStorage::read()로 파트에서 데이터 읽기
   └── Block 단위로 필터링, 집계, 정렬 등 처리

7. 결과 출력
   └── Format Processor → WriteBuffer → 클라이언트

8. 정리
   └── slot 반환 → ConcurrencyControl이 대기 중인 다른 쿼리에 재분배
   └── ThreadStatus 소멸, ProfileEvents 기록
   └── system.query_log에 로그 기록
```

---

## 3.6 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Context 계층** | 6단계 설정 머지. Background 작업은 Session/Query 설정 무시 |
| **Thread Pool 계층** | 모든 특수 풀은 Global pool 위에 구축. IO 풀은 CPU 풀 고갈 방지용 격리 |
| **MergeTreeBackgroundExecutor** | 머지/뮤테이션 전담. 선점 가능한 스텝 기반 스케줄링으로 짧은 작업 우선 |
| **ThreadStatus + Thread Group** | 쿼리 단위 메모리·성능 추적의 기반. system.query_log의 데이터 소스 |
| **ConcurrencyControl** | CPU slot 기반. 모든 쿼리에 최소 1스레드 보장(soft limit). 런타임 조정 가능 |
| **Slot 상태 머신** | free → granted → acquired. 점진적 스케일업 지원 |

---

## 3.7 운영 시 주의점

### Thread Pool 관련

- **`max_thread_pool_size` 모니터링**: Global pool이 고갈되면 모든 하위 풀이 영향을 받는다. `system.metrics`에서 `GlobalThread`, `GlobalThreadActive` 메트릭을 관찰하라.
- **IO pool 분리의 중요성**: S3 기반 스토리지를 사용한다면 `max_io_thread_pool_size`와 `max_backups_io_thread_pool_size`를 독립적으로 튜닝하라. 백업이 인터랙티브 쿼리의 IO 스레드를 잡아먹을 수 있다.
- **Server pool(`max_connection`)**: 동시 커넥션 수 상한이다. 커넥션 풀을 사용하는 애플리케이션에서는 이 값이 병목이 되지 않도록 확인하라.

### ConcurrencyControl 관련

- **`concurrent_threads_soft_limit_num` 설정**: 기본값은 0(제한 없음)이다. 동시 쿼리가 많은 환경에서는 CPU 코어 수의 1~2배로 설정하는 것을 고려하라. 너무 낮으면 개별 쿼리 성능이 저하되고, 너무 높거나 0이면 컨텍스트 스위칭 비용이 증가한다.
- **Soft limit의 의미**: 모든 쿼리가 최소 1개 스레드를 보장받으므로, 동시 쿼리 수가 slot 총 수를 초과할 수 있다. 이는 의도된 설계다. "쿼리가 실행조차 안 되는" 상황은 발생하지 않는다.
- **런타임 조정**: 서버 재시작 없이 slot 수를 변경할 수 있으므로, 피크 시간대에 동적으로 조정하는 운영 패턴이 가능하다.

### Background 설정 관련

- **Background 프로파일 확인**: 머지가 느리거나 뮤테이션이 밀리면 `background` 프로파일의 설정을 확인하라. `SET`으로는 변경되지 않는다.
- **`background_profile` 오버라이드**: 기본 `'background'` 대신 커스텀 프로파일을 지정하여, 프로덕션과 백그라운드 작업의 리소스를 명시적으로 분리할 수 있다.

### 모니터링 쿼리 예시

```sql
-- 현재 활성 스레드 현황
SELECT metric, value FROM system.metrics
WHERE metric IN ('GlobalThread', 'GlobalThreadActive',
                 'IOThreads', 'IOThreadsActive',
                 'BackgroundMergesAndMutationsPoolTask',
                 'BackgroundSchedulePoolTask');

-- 쿼리별 메모리 사용량 (Thread Group 기반 집계)
SELECT query_id, memory_usage, peak_memory_usage,
       ProfileEvents['RealTimeMicroseconds'] AS cpu_time_us
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY peak_memory_usage DESC
LIMIT 10;

-- ConcurrencyControl 영향 확인: 쿼리별 실제 스레드 수
SELECT query_id, Settings['max_threads'] AS requested_threads,
       length(thread_ids) AS actual_threads
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 20;
```

---

*다음 장: 4장. Table Parts — MergeTree 스토리지의 물리적 구조*
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
# 6장. Table Part Merges — 백그라운드 머지의 동작 원리

1장에서 ClickHouse의 핵심 설계 원칙으로 **Merge-time Computation**을 소개했다. INSERT를 가볍게 유지하고, 무거운 데이터 처리를 백그라운드 머지로 미루는 전략이다. 이 장에서는 그 머지 프로세스의 구체적인 동작 원리를 파고든다.

---

## 6.1 왜 머지가 필요한가

4장에서 배웠듯이, INSERT마다 새로운 immutable 파트가 생성된다. 이를 방치하면 파트 수가 한없이 늘어나고, SELECT 쿼리는 수천 개의 파트를 모두 열어 읽어야 한다. 파트 수를 제어하고, 데이터 최적화를 수행하기 위해 백그라운드 머지가 필수적이다.

공식 merges 문서의 요약:

> ① Inserts create sorted, immutable data parts.
> ② All data processing is offloaded to background part merges.
> This makes data writes lightweight and highly efficient.

---

## 6.2 머지의 기본 메커니즘

ClickHouse는 파티션별로 작은 파트들을 지속적으로 합쳐 큰 파트로 만든다. 압축 크기 약 **~150GB** (`max_bytes_to_merge_at_max_space_in_pool`)에 도달하면 더 이상 머지 대상이 되지 않는다.

### 머지의 4단계

```
① Decompression & Loading
   머지 대상 파트들의 압축된 column.bin 파일을 압축 해제하고 메모리에 로드

② Merging
   데이터를 하나의 큰 컬럼 파일로 병합 (sorting key 순서 유지)
   → 엔진 종류에 따라 추가 처리 (중복 제거, 집계 등)

③ Indexing
   병합된 컬럼 파일에 대해 새로운 sparse primary index 생성

④ Compression & Storage
   새 컬럼 파일과 인덱스를 압축하여 새 디렉터리(= 새 파트)에 저장
```

추가로 secondary skipping indexes, column statistics, checksums, min-max indexes 등의 메타데이터도 머지된 컬럼 파일 기반으로 재생성된다.

### 머지 후 정리

머지로 생성된 원본 파트들은 **inactive**로 마킹되고, `old_parts_lifetime` (기본 8분) 후 삭제된다. 시간이 지남에 따라 머지의 계층 구조가 형성되며, 이것이 **MergeTree**라는 이름의 유래다.

---

## 6.3 Merge Level: 파트의 이력 추적

파트가 머지를 거칠 때마다 **merge level**이 1씩 증가한다.

```
level 0: INSERT로 직접 생성된 파트 (머지 안 됨)
level 1: level 0 파트들이 1회 머지된 결과
level 2: level 1 파트들이 다시 머지된 결과
...
```

실제 관찰 예:

```sql
-- 초기 상태: 24개 파트가 4개로 머지됨 (level 1)
   ┌─name────────┬─level─┬────rows─┐
1. │ all_0_5_1   │     1 │ 6368414 │
2. │ all_12_17_1 │     1 │ 6442494 │
3. │ all_18_23_1 │     1 │ 5977762 │
4. │ all_6_11_1  │     1 │ 6459763 │
   └─────────────┴───────┴─────────┘

-- 시간 경과 후: 4개 파트가 1개로 최종 머지 (level 2)
   ┌─name───────┬─level─┬─────rows─┐
1. │ all_0_23_2 │     2 │ 25248433 │
   └────────────┴───────┴──────────┘
```

추가 INSERT가 없으면 결국 하나의 파트로 수렴한다 (파티셔닝이 없는 경우).

---

## 6.4 동시 머지(Concurrent Merges)

단일 ClickHouse 서버는 **여러 백그라운드 머지 스레드**를 사용해 동시에 머지를 실행한다. 스레드 수는 `background_pool_size` 설정으로 조절된다.

각 머지 스레드의 실행 루프:

```
loop {
  ① 다음 머지할 파트 결정 → 메모리에 로드
  ② 메모리에서 파트들을 하나의 큰 파트로 병합
  ③ 병합된 파트를 디스크에 기록
}
```

CPU 코어와 RAM을 늘리면 백그라운드 머지 처리량을 높일 수 있다.

---

## 6.5 메모리 최적화 머지 (Vertical Merging)

기본적으로 머지할 파트들을 한꺼번에 메모리에 로드하지만, 파트가 크면 메모리 소비가 과도해질 수 있다. 이를 위해 **vertical merging** 모드가 존재한다.

Vertical merging은 파트 전체를 한 번에 로드하는 대신, **블록 청크 단위로 분할하여 로드하고 머지**한다. 메모리 소비를 줄이는 대신 머지 속도가 다소 느려지는 트레이드오프다.

활성화 여부는 파트 크기와 설정에 따라 자동 결정된다.

---

## 6.6 엔진별 머지 동작: Merge-time Computation의 실체

머지의 ② 단계(Merging)는 테이블 엔진에 따라 **근본적으로 다른 일**을 한다. 이것이 "모든 데이터 처리를 백그라운드 머지로 이관한다"는 원칙의 실체다.

### Standard MergeTree: 정렬 병합

가장 기본. 여러 파트의 정렬된 컬럼들을 **sorting key 순서를 유지하며 하나로 합친다** (merge sort). 데이터 자체의 변환은 없다.

```
Part A (sorted by town, street): [Aberdeen/High St, London/Baker St, ...]
Part B (sorted by town, street): [Bath/King Rd, London/Oxford St, ...]
                    ↓ merge (maintaining sort order)
Merged Part: [Aberdeen/High St, Bath/King Rd, London/Baker St, London/Oxford St, ...]
```

### ReplacingMergeTree: 중복 제거 병합

Standard 머지와 동일하게 정렬 병합하되, **같은 sorting key를 가진 행 중 가장 최신 것만 유지**한다. 나머지는 버린다.

```
Part A: [(London, Baker St, id=1, ver=1), ...]
Part B: [(London, Baker St, id=1, ver=2), ...]
                    ↓ replacing merge
Merged:  [(London, Baker St, id=1, ver=2)]   ← 최신 버전만 유지
```

"최신"의 기준은 해당 행이 포함된 파트의 생성 타임스탬프, 또는 명시적 version 컬럼이다.

**중요한 주의**: 이것은 "진짜 UPDATE"가 아니다. 머지가 언제 실행될지 사용자가 제어할 수 없고, 데이터는 대부분 여러 파트에 걸쳐 존재하므로, **머지 전에는 중복 행이 보인다**. 최종적으로 정확한 결과가 필요하면 `FINAL` 키워드를 사용하거나, 쿼리에서 직접 중복을 처리해야 한다.

### SummingMergeTree: 합산 병합

같은 sorting key를 가진 모든 행을 **하나의 행으로 대체**하며, 숫자형 컬럼의 값을 **합산**한다.

```
Part A: [(London, price=100), (London, price=200)]
Part B: [(London, price=150)]
                    ↓ summing merge
Merged:  [(London, price=450)]
```

### AggregatingMergeTree: 범용 집계 병합

SummingMergeTree의 일반화. 90개 이상의 집계 함수를 머지 시점에 적용할 수 있다. **부분 집계 상태(partial aggregation state)**를 저장하고, 머지 시 이를 결합한다.

```
Part A: [(London, avg_state={sum=300, count=2})]
Part B: [(London, avg_state={sum=150, count=1})]
                    ↓ aggregating merge
Merged:  [(London, avg_state={sum=450, count=3})]   → avg = 150
```

부분 집계 상태를 사용하므로 단순 합산이 불가능한 함수(avg, uniq 등)도 정확하게 증분 처리할 수 있다.

### CollapsingMergeTree: 부호 기반 병합

`sign` 컬럼(1 또는 -1)을 사용해 행의 삽입/취소를 표현한다. 머지 시 같은 sorting key의 +1과 -1 행이 **서로 상쇄**된다.

---

## 6.7 "머지 시점은 사용자가 제어할 수 없다"

이것은 운영에서 반드시 이해해야 하는 점이다. 공식 architecture.md:

> "Keep in mind that these are not real updates because users usually have no control over the time when background merges are executed, and data in a MergeTree table is almost always stored in more than one part, not in completely merged form."

ReplacingMergeTree, SummingMergeTree 등은 "최종 상태"를 보장하기 위해 머지에 의존하지만, 머지 시점은 ClickHouse 내부 스케줄러가 결정한다. 따라서:

- SELECT 결과에 머지 전의 중복/미집계 데이터가 포함될 수 있다
- `SELECT ... FINAL`은 쿼리 시점에 머지 로직을 강제 적용하지만, 성능이 저하된다
- 프로덕션에서는 쿼리 레벨에서 중복/집계를 처리하는 패턴이 더 일반적이다

---

## 6.8 OPTIMIZE TABLE: 수동 머지 트리거

```sql
-- 하나의 머지 사이클 실행
OPTIMIZE TABLE mydb.mytable;

-- 모든 파트를 하나로 최종 머지 (매우 무거움)
OPTIMIZE TABLE mydb.mytable FINAL;

-- 특정 파티션만 최종 머지
OPTIMIZE TABLE mydb.mytable PARTITION '2024-01-01' FINAL;
```

**`OPTIMIZE FINAL`은 프로덕션에서 극히 주의해서 사용해야 한다.** 공식 best-practices에서 별도 문서로 "Avoid OPTIMIZE FINAL"을 다룰 만큼 강조하는 사항이다. 전체 테이블 데이터를 다시 쓰므로 막대한 I/O와 CPU를 소비한다.

---

## 6.9 머지와 Write Amplification

4장에서 언급한 write amplification을 수치로 이해하자.

```
INSERT 1회: 데이터 1번 기록
  → level 0 → level 1 머지: 데이터 2번째 기록
    → level 1 → level 2 머지: 데이터 3번째 기록
      → ...
```

ClickHouse 24.10부터 내장 모니터링 대시보드(`/merges` HTTP handler)에서 write amplification을 시각적으로 추적할 수 있다. 대시보드는 다음을 보여준다:

1. 활성 파트 수 변화
2. 파트 머지의 시각적 표현 (박스 크기 = 파트 크기)
3. Write amplification 수치

---

## 6.10 모니터링

### 진행 중인 머지 확인

```sql
SELECT
    database,
    `table`,
    partition_id,
    round(progress * 100, 1) AS progress_pct,
    num_parts,
    formatReadableSize(total_size_bytes_compressed) AS total_size,
    formatReadableTimeDelta(elapsed) AS elapsed,
    merge_type,
    merge_algorithm
FROM system.merges
ORDER BY elapsed DESC;
```

### 머지 처리량과 지연 추적

```sql
-- 최근 머지 이력 (query_log에서)
SELECT
    event_time,
    duration_ms,
    read_rows,
    written_rows,
    formatReadableSize(read_bytes) AS read_size,
    formatReadableSize(written_bytes) AS written_size
FROM system.part_log
WHERE event_type = 'MergeParts'
  AND database = 'mydb'
  AND `table` = 'mytable'
ORDER BY event_time DESC
LIMIT 20;
```

### 머지 지연 경보: 파트 수 추이

```sql
-- 파트 수가 증가 추세인 테이블 식별
SELECT
    database,
    `table`,
    count() AS active_parts,
    max(modification_time) AS last_insert,
    min(level) AS min_level  -- level 0 파트가 많으면 머지 지연
FROM system.parts
WHERE active
GROUP BY database, `table`
HAVING active_parts > 100
ORDER BY active_parts DESC;
```

---

## 6.11 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **머지의 목적** | 파트 수 제어 + 엔진별 데이터 처리(중복 제거, 집계 등) |
| **머지 4단계** | 압축 해제·로드 → 병합 → 인덱싱 → 압축·저장 |
| **엔진별 차이** | MergeTree(정렬 병합), Replacing(최신 유지), Summing(합산), Aggregating(부분 집계 상태) |
| **동시 머지** | `background_pool_size` 스레드가 병렬 실행. CPU/RAM이 처리량 결정 |
| **Vertical merging** | 메모리 부족 시 블록 청크 단위로 분할 머지 (속도 vs 메모리 트레이드오프) |
| **머지 시점** | 사용자가 제어 불가. 머지 전에는 중복/미집계 데이터 존재 가능 |
| **Merge level** | 머지 횟수 추적. level 0이 많으면 머지 지연 징후 |

---

## 6.12 운영 시 주의점

### 머지 스레드 튜닝

- `background_pool_size`: 머지/뮤테이션 병렬 스레드 수. 기본값은 CPU 코어 수의 절반 수준. INSERT 속도가 머지보다 빠르면 증가 고려.
- 3장에서 다룬 대로, 이 설정은 Background context에서 관리되므로 `SET`이 아닌 config 파일이나 background 프로파일에서 변경해야 한다.

### Replacing/Aggregating 엔진 사용 시

- **`FINAL` 남용 금지**: 쿼리마다 머지 로직을 강제하면 성능 심각 저하. 대안으로 `argMax`, `groupBy`를 활용한 쿼리 패턴 사용.
- **머지 완료 전 데이터 읽기**: 비즈니스 로직이 "최신 상태"를 요구하면, 쿼리에서 직접 중복 처리를 구현하라. 머지에 의존하면 타이밍에 따라 결과가 달라진다.

### 디스크 공간

- 머지 중 원본 + 결과 파트가 동시 존재. 대형 테이블에서는 일시적으로 **2배 공간** 필요.
- `system.merges`에서 진행 중인 머지의 `total_size_bytes_compressed`를 확인해 디스크 여유를 점검하라.

### 머지가 밀릴 때

원인 순서대로 확인:
1. INSERT 속도가 머지 속도를 초과 → 배치 크기 증가 또는 `background_pool_size` 증가
2. 디스크 I/O 병목 → 디스크 처리량 확인 (특히 HDD 환경)
3. 파티셔닝 과다 → 파티션당 파트 수 확인 (5장 참조)
4. Mutation 진행 중 → `system.mutations`에서 `is_done = 0` 확인. Mutation이 머지 스레드를 점유

---

*다음 장: 7장. Primary Index — Sparse Index의 원리와 ORDER BY 키 설계*
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
# 8장. Shards와 Replicas — 분산 아키텍처

단일 서버의 한계에 도달하면 ClickHouse 클러스터를 구성한다. 이 장에서는 Shard(데이터 수평 분할), Replica(데이터 복제), Distributed 테이블(라우팅 레이어), 그리고 분산 쿼리 실행의 구조와 한계를 다룬다.

---

## 8.1 Shard: 데이터를 나누는 단위

### 샤딩이 필요한 두 가지 경우

1. **데이터가 단일 서버의 스토리지를 초과**할 때
2. **단일 서버의 처리 성능이 부족**할 때

샤딩은 테이블의 데이터를 여러 ClickHouse 서버로 **수평 분할**한다. 각 shard는 데이터의 부분 집합을 담는 일반적인 ClickHouse 테이블이며, 독립적으로 쿼리할 수 있다.

```
[전체 데이터]
   ├── Shard 1 (Server A): 행 1 ~ 1000만
   ├── Shard 2 (Server B): 행 1000만+1 ~ 2000만
   └── Shard 3 (Server C): 행 2000만+1 ~ 3000만
```

### Distributed 테이블: 데이터를 저장하지 않는 라우팅 레이어

개별 shard에 직접 쿼리하면 부분 데이터만 보인다. 전체 데이터셋에 대한 통합 뷰를 제공하는 것이 **Distributed 테이블**이다.

```sql
CREATE TABLE uk.uk_price_paid_simple_dist ON CLUSTER test_cluster
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = Distributed('test_cluster', 'uk', 'uk_price_paid_simple', rand())
```

핵심 특성:
- **데이터를 자체 저장하지 않는다** — 모든 로컬 테이블에 대한 "view"일 뿐
- **SELECT**: 모든 shard에 쿼리를 전달하고, 결과를 수집·병합
- **INSERT**: sharding key를 평가하여 적절한 shard로 행을 라우팅

### Sharding Key와 INSERT 라우팅

```
INSERT → Distributed 테이블
  ↓
각 행에 대해:
  ① sharding key 평가 (예: rand())
  ② 결과 % shard 수 = 대상 서버 ID
  ③ 해당 서버의 로컬 테이블에 INSERT
```

`rand()`는 무작위 분산을 보장한다. 하지만 특정 쿼리 패턴에 따라 `cityHash64(user_id)` 같은 결정적 키를 사용하면, 같은 사용자의 데이터가 같은 shard에 모여 JOIN이나 GROUP BY 효율이 높아질 수 있다.

---

## 8.2 분산 쿼리 실행

### SELECT 처리 흐름

```
① 클라이언트 → Distributed 테이블에 SELECT 전송

② Distributed 테이블이 모든 shard에 쿼리 전달
   → 각 서버가 로컬에서 독립적으로 쿼리 실행 (병렬)

③ Initiator 서버가 모든 로컬 결과를 수집

④ 결과를 병합하여 최종 글로벌 결과 생성

⑤ 클라이언트에 반환
```

공식 architecture.md의 설명:

> "The Distributed table requests remote servers to process a query just up to a stage where intermediate results from different servers can be merged. Then it receives the intermediate results and merges them."

핵심 설계 원칙: **가능한 많은 작업을 원격 서버에서 처리**하고, 네트워크로 전송하는 중간 데이터를 최소화한다.

### Global Query Plan이 없다

이것은 ClickHouse 분산 쿼리의 가장 중요한 구조적 제약이다:

> "There is no global query plan for distributed query execution. Each node has its local query plan for its part of the job. We only have simple one-pass distributed query execution: we send queries for remote nodes and then merge the results."

**1-pass 분산 실행**만 지원한다: 쿼리를 원격 노드에 보내고 결과를 머지. 그 이상의 조율은 없다.

### 분산 실행이 어려운 쿼리

다음 유형의 쿼리에서 이 제약이 문제가 된다:

**고카디널리티 GROUP BY**: 각 shard의 로컬 GROUP BY 결과가 방대하면, Initiator가 이를 머지할 때 메모리와 네트워크 부하가 급증한다.

**대용량 데이터 JOIN**: 두 개의 Distributed 테이블을 JOIN하면, 한쪽 테이블의 데이터를 다른쪽의 모든 shard에 브로드캐스트해야 할 수 있다.

**서브쿼리의 Distributed 테이블**: `IN` 또는 `JOIN` 절에 Distributed 테이블이 포함되면 실행 전략이 복잡해진다.

공식 문서는 이에 대해 솔직하다:

> "In such cases, we need to 'reshuffle' data between servers, which requires additional coordination. ClickHouse does not support that kind of query execution, and we need to work on it."

---

## 8.3 Replica: 데이터 복제

### 복제의 목적

1. **데이터 무결성 (Data Integrity)**: 하드웨어 장애 시 데이터 손실 방지
2. **페일오버 (Failover)**: 서버 장애 시 다른 레플리카에서 서비스 지속
3. **쿼리 처리량 (Throughput)**: 여러 레플리카에 쿼리를 분산하여 병렬 처리

### 복제의 구조

```
Cluster
  ├── Shard 1
  │     ├── Replica A (Server 1)
  │     ├── Replica B (Server 2)
  │     └── Replica C (Server 3)
  └── Shard 2
        ├── Replica A (Server 4)
        ├── Replica B (Server 5)
        └── Replica C (Server 6)
```

각 shard 내에서 모든 레플리카는 **동일한 데이터**를 보유한다. SELECT 쿼리 시 Distributed 테이블은 각 shard에서 **하나의 레플리카만 선택**하여 쿼리를 전달한다.

### ReplicatedMergeTree

복제는 `ReplicatedMergeTree` 스토리지 엔진으로 구현된다:

```sql
CREATE TABLE uk.uk_price_paid_simple ON CLUSTER test_cluster
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/uk_price_paid', '{replica}')
ORDER BY (town, street);
```

**같은 ZooKeeper 경로를 가진 모든 테이블이 서로의 레플리카**가 된다. 레플리카는 테이블 생성/삭제만으로 동적으로 추가·제거할 수 있다.

### 비동기 멀티마스터 복제

```
             ZooKeeper / ClickHouse Keeper
                    │
    ┌───────────────┼───────────────┐
    │               │               │
  Replica A      Replica B      Replica C
  (INSERT 가능)  (INSERT 가능)  (INSERT 가능)
```

핵심 특성:

**비동기(Asynchronous)**: 데이터가 어느 레플리카에든 INSERT된 후, 다른 레플리카에 비동기적으로 복제된다. INSERT 시점에 모든 레플리카의 확인을 기다리지 않는다.

**멀티마스터(Multi-master)**: ZooKeeper 세션이 있는 **어떤 레플리카에든 INSERT 가능**하다. 단일 마스터 병목이 없다.

**Conflict-free**: ClickHouse는 UPDATE를 지원하지 않으므로 (INSERT only + 머지 기반 변환), 복제 시 충돌이 발생하지 않는다.

### 복제 로그와 물리적 복제

ZooKeeper에 저장되는 **복제 로그(replication log)**가 복제를 조율한다. 로그에 기록되는 액션:

- `get part`: 파트 다운로드
- `merge parts`: 파트 머지
- `drop partition`: 파티션 삭제

각 레플리카는 복제 로그를 자신의 큐에 복사하고 순서대로 실행한다.

**물리적 복제(Physical Replication)**: 쿼리가 아닌 **압축된 파트 자체**가 노드 간 전송된다. 이는 네트워크 효율을 위한 설계다. 머지는 대부분의 경우 **각 레플리카에서 독립적으로 수행**되어 네트워크 증폭(network amplification)을 회피한다. 대용량 머지된 파트는 복제 지연(replication lag)이 심각할 때만 네트워크로 전송된다.

### 머지 조율

모든 파트가 모든 레플리카에서 **byte-identical**하게 머지되도록 조율한다:

1. **리더(leader)** 중 하나가 새 머지를 시작하고 복제 로그에 "merge parts" 액션을 기록
2. 다른 레플리카가 로그를 읽고 동일한 머지를 독립 수행
3. 결과가 byte-identical이므로 네트워크 전송 불필요

여러 레플리카(또는 전부)가 동시에 리더가 될 수 있다. `replicated_can_become_leader` 설정으로 특정 레플리카의 리더 역할을 방지할 수 있다.

### 일관성 복구

각 레플리카는 ZooKeeper에 자신의 상태(파트 목록 + 체크섬)를 저장한다.

```
로컬 파일시스템 상태 ≠ ZooKeeper 참조 상태
  → 누락/손상 파트를 다른 레플리카에서 다운로드하여 복구
  → 예상치 못한/손상된 데이터는 별도 디렉터리로 이동 (삭제하지 않음)
```

---

## 8.4 클러스터의 구조적 한계

공식 architecture.md의 경고:

> "The ClickHouse cluster consists of independent shards, and each shard consists of replicas. The cluster is **not elastic**, so after adding a new shard, data is not rebalanced between shards automatically."

### 핵심 한계 정리

**자동 리밸런싱 없음**: 새 shard를 추가해도 기존 데이터가 자동으로 재분배되지 않는다. 수동으로 데이터를 이동해야 한다.

**노드 간 데이터 재분배(reshuffle) 미지원**: 복잡한 분산 JOIN이나 고카디널리티 GROUP BY에서 노드 간 데이터 교환이 필요하지만, 현재 지원하지 않는다.

**수십 노드까지는 괜찮으나 수백 노드에서는 문제**: 공식 문서가 직접 인정하는 확장성 한계. 동적으로 복제 영역을 분할하고 밸런싱하는 테이블 엔진이 필요하다고 명시.

공식 문서 원문:

> "This implementation gives you more control, and it is ok for relatively small clusters, such as tens of nodes. But for clusters with hundreds of nodes that we are using in production, this approach becomes a significant drawback."

### ClickHouse Cloud의 접근

ClickHouse Cloud에서는 이 한계를 다른 아키텍처로 해결한다:
- **SharedMergeTree**: 오브젝트 스토리지 기반으로 레플리카를 대체하여 고가용성 확보
- **Parallel Replicas**: 전통적 shared-nothing 클러스터의 shard와 유사하게 동작

---

## 8.5 ClickHouse Keeper

ZooKeeper의 ClickHouse 네이티브 대체품이다. 복제와 분산 DDL에 필수적이며, 복제 로그 저장, 리더 선출, 분산 DDL 조율을 담당한다.

Keeper가 필요한 경우:
- `ReplicatedMergeTree` 사용 시
- `ON CLUSTER` 분산 DDL 사용 시
- 분산 `INSERT ... SELECT` 사용 시

---

## 8.6 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Shard** | 데이터를 수평 분할하는 단위. 각 shard는 독립적 테이블 |
| **Distributed 테이블** | 데이터를 저장하지 않는 라우팅 레이어. SELECT 전달 + INSERT 라우팅 |
| **분산 SELECT** | 모든 shard에 쿼리 전달 → 로컬 실행 → Initiator에서 결과 병합 |
| **Global query plan 없음** | 1-pass 실행만 지원. 고카디널리티 GROUP BY, 대용량 JOIN에 한계 |
| **Replica** | shard 데이터의 복제본. 비동기 멀티마스터. conflict-free |
| **물리적 복제** | 쿼리가 아닌 압축 파트 전송. 머지는 각 레플리카에서 독립 수행 |
| **ZooKeeper/Keeper** | 복제 로그, 리더 선출, 일관성 복구의 중앙 조율 |
| **Not elastic** | 자동 리밸런싱 없음. 수십 노드까지 적합 |

---

## 8.7 운영 시 주의점

### Sharding Key 선택

- **`rand()`**: 균등 분산 보장. 대부분의 경우 적합. 단, 같은 엔티티의 데이터가 여러 shard에 분산되어 JOIN 비효율
- **결정적 키** (`cityHash64(user_id)` 등): 같은 사용자 데이터가 같은 shard에 모임. 특정 shard에 데이터 쏠림(skew) 위험
- **데이터 쏠림 모니터링**: shard 간 데이터 크기가 크게 차이나면 쿼리 성능이 불균일해진다

```sql
-- shard별 데이터 분포 확인
SELECT
    hostName() AS host,
    count() AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM clusterAllReplicas('test_cluster', system.parts)
WHERE database = 'uk' AND `table` = 'uk_price_paid_simple' AND active
GROUP BY host;
```

### INSERT Quorum

```sql
-- 기본: quorum 없음 → 노드 장애 시 방금 넣은 데이터 유실 가능
-- quorum 활성화: N개 레플리카에 확인 후 INSERT 완료
SET insert_quorum = 2;
INSERT INTO ...;
```

데이터 유실이 허용되지 않는 환경에서는 `insert_quorum` 설정을 활성화하라. 대신 INSERT 레이턴시가 증가한다.

### 복제 지연(Replication Lag) 모니터링

```sql
SELECT
    database,
    `table`,
    replica_name,
    is_leader,
    absolute_delay,
    queue_size,
    inserts_in_queue,
    merges_in_queue
FROM system.replicas
WHERE absolute_delay > 0
ORDER BY absolute_delay DESC;
```

`absolute_delay`가 증가 추세면 레플리카가 쓰기를 따라잡지 못하는 것이다. 네트워크 대역폭, 디스크 I/O, Keeper 상태를 확인하라.

### Distributed 테이블 INSERT 주의

Distributed 테이블에 INSERT하면 데이터가 Initiator 서버에 임시 저장된 후 비동기로 shard에 전달된다. Initiator 장애 시 이 임시 데이터가 유실될 수 있다.

**권장**: 가능하면 **로컬 테이블(shard)에 직접 INSERT**하라. 애플리케이션 레벨에서 sharding key를 계산하여 올바른 서버에 직접 전송하는 것이 가장 안전하다.

### Keeper 운영

- **홀수 노드** (3, 5, 7)로 Keeper 클러스터 구성 (과반수 합의를 위해)
- Keeper 장애 시 INSERT와 머지가 중단되지만, **SELECT는 정상 동작** (읽기는 Keeper에 의존하지 않음)
- Keeper 상태 모니터링: `system.zookeeper` 테이블, `four_letter_word_white_list` 커맨드

---

*다음 장: 9장~11장은 운영 실무 파트로, INSERT 전략, 피해야 할 패턴, 모니터링과 트러블슈팅을 다룬다.*
# 9장. INSERT 전략 선택

이 장부터 운영 실무를 다룬다. 효율적인 데이터 적재(ingestion)는 ClickHouse 운영의 출발점이며, 잘못된 INSERT 전략은 "Too many parts"부터 데이터 유실까지 다양한 문제를 유발한다.

---

## 9.1 동기 INSERT (기본)

ClickHouse의 기본 INSERT는 **동기적**이다. 각 INSERT 문이 즉시 디스크에 하나의 파트를 생성하며, 메타데이터와 인덱스를 포함한다.

### 배칭이 핵심이다

4장에서 배운 대로, INSERT마다 파트가 생성되므로 **클라이언트 측 배칭**이 필수다. 공식 best-practices의 첫 번째 결정이 배치 크기다.

권장 사항:
- 최소 **20,000행** 이상을 한 번에 INSERT
- 이상적으로는 수만~수십만 행 단위
- 초당 수천 번의 소량 INSERT는 MergeTree에 최적이 아님

### 멱등한 재시도 (Idempotent Retries)

MergeTree 엔진은 기본적으로 INSERT를 **자동 중복 제거**한다. 다음 상황에서 안전한 재시도가 가능하다:

- INSERT가 성공했으나 네트워크 중단으로 클라이언트가 확인을 받지 못한 경우
- INSERT가 서버 측에서 실패하고 타임아웃된 경우

재시도 시 **배치 내용과 순서를 동일하게 유지**해야 중복 제거가 작동한다.

### INSERT 대상 선택

| 대상 | 장점 | 단점 |
|------|------|------|
| **로컬 테이블 직접 INSERT** | 가장 효율적. `internal_replication=true` 시 복제 자동 | 클라이언트가 shard별 로드밸런싱 필요 |
| **Distributed 테이블 INSERT** | 클라이언트가 어떤 노드에든 전송 가능 | Initiator에 임시 저장 후 비동기 전달. Initiator 장애 시 유실 위험 |

**권장**: 가능하면 로컬 테이블에 직접 INSERT.

---

## 9.2 비동기 INSERT

소량 빈번 INSERT가 불가피한 환경(IoT, observability, 에이전트 기반 수집 등)에서는 **async insert** 모드를 사용한다.

```sql
SET async_insert = 1;
```

동작 원리: ClickHouse 서버가 여러 소량 INSERT를 **서버 측에서 버퍼링**하고, 버퍼 크기 임계값 또는 타임아웃 초과 시 하나의 파트로 생성한다.

이점:
- 클라이언트가 배칭을 구현하지 않아도 됨
- 파트 수 폭증 방지
- INSERT 레이턴시는 증가할 수 있으나 처리량은 유지

---

## 9.3 포맷 선택

| 포맷 | 특성 | 권장 사용처 |
|------|------|------------|
| **Native** | 가장 효율적. 컬럼 지향, 서버 파싱 최소. Go/Python 클라이언트 기본값 | 고처리량 애플리케이션 |
| **RowBinary** | 효율적 행 기반. Java 클라이언트 기본값 | 클라이언트에서 컬럼 변환 어려울 때 |
| **JSONEachRow** | 사용 쉬움. 파싱 비용 높음 | 저볼륨, 빠른 통합 |

### 압축

전송 전 압축은 네트워크 오버헤드를 줄이고 INSERT 속도를 높인다:

- **LZ4** (기본): 빠르고 가벼움. 대부분의 경우 최적
- **ZSTD**: 높은 압축률이지만 CPU 소비 더 큼. 대역폭 제약이나 egress 비용이 클 때

Native 포맷 + LZ4 압축이 최고의 ingestion 성능을 제공한다.

### 사전 정렬 (Pre-sort)

클라이언트에서 프라이머리 키 순서로 데이터를 정렬하여 보내면, 서버의 정렬 단계를 skip하거나 간소화할 수 있다. 압축 효율도 향상된다.

단, **선택적 최적화**이다. ClickHouse의 서버 측 병렬 정렬이 이미 매우 빠르므로, 데이터가 이미 거의 정렬되어 있거나 클라이언트 리소스가 여유로울 때만 권장된다.

---

## 9.4 인터페이스 선택: Native vs HTTP

| | Native | HTTP |
|---|--------|------|
| **포맷** | 항상 Native (최고 효율) | 70+ 포맷 지원 |
| **압축** | 블록 단위 LZ4/ZSTD | 전체 페이로드 단위 |
| **클라이언트 최적화** | MATERIALIZED/DEFAULT 값 클라이언트 계산 가능 | 불가 |
| **로드밸런싱** | 어려움 | 쉬움 (표준 HTTP) |
| **권장** | 최대 성능이 필요한 고처리량 ingestion | 대부분의 애플리케이션 (유연성 + 호환성) |

---

## 9.5 Key Takeaways

| 항목 | 핵심 |
|------|------|
| 동기 INSERT | 배칭 필수 (20,000행 이상). 멱등 재시도 지원 |
| 비동기 INSERT | 소량 빈번 INSERT 환경의 대안. 서버 측 버퍼링 |
| 포맷 | Native > RowBinary > JSONEachRow (성능 순) |
| 압축 | LZ4 기본. 대역폭 제약 시 ZSTD |
| 인터페이스 | 성능: Native / 유연성: HTTP |
| INSERT 대상 | 로컬 테이블 직접 INSERT 권장. Distributed INSERT는 유실 위험 |

---

---

# 10장. 피해야 할 패턴들

이 장에서는 ClickHouse 운영에서 자주 발생하는 안티패턴들을 정리한다. 각각이 왜 문제인지, 어떤 대안이 있는지를 다룬다.

---

## 10.1 Mutation (ALTER TABLE UPDATE/DELETE) 남용

### 왜 문제인가

ClickHouse의 파트는 immutable이다. `ALTER TABLE UPDATE` 또는 `ALTER TABLE DELETE`는 대상 파트 전체를 다시 쓴다. 이것은:

- 막대한 디스크 I/O와 CPU를 소비한다
- 백그라운드 머지 스레드를 점유한다 (정상 머지 지연)
- 완료까지 오래 걸릴 수 있다 (비동기 실행)

### 대안

- **ReplacingMergeTree**: 최신 버전만 유지하는 자연스러운 "UPDATE" 패턴
- **CollapsingMergeTree**: +1/-1 sign으로 삽입/취소를 표현
- **경량 DELETE** (`DELETE FROM`): ClickHouse 23.3+에서 지원. 파트를 다시 쓰지 않고 마스크로 삭제 표시
- 설계 시점에서 UPDATE가 불필요하도록 스키마를 모델링하라

---

## 10.2 OPTIMIZE FINAL 남용

### 왜 문제인가

`OPTIMIZE TABLE ... FINAL`은 테이블의 **모든 파트를 단일 파트로 강제 머지**한다. 전체 데이터를 다시 쓰므로:

- 디스크 I/O가 테이블 크기에 비례하여 폭증
- 진행 중인 다른 머지를 방해
- 프로덕션 쿼리 성능에 심각한 영향

### 대안

- **백그라운드 머지에 맡겨라**: ClickHouse가 자동으로 최적의 시점에 머지한다
- ReplacingMergeTree에서 최신 상태가 필요하면 `SELECT ... FINAL`을 사용하되, 이것도 쿼리별 비용이 크므로 `argMax` 패턴이 더 나을 수 있다
- 특정 파티션만 머지가 필요하면 `OPTIMIZE TABLE ... PARTITION ... FINAL`로 범위를 제한하라

---

## 10.3 과도한 파티셔닝

5장에서 상세히 다루었으므로 핵심만 반복한다:

- 파티션 수 **1,000~10,000 이하** 유지
- 파티셔닝은 **데이터 관리 기법**이지 쿼리 최적화 도구가 아니다
- 파티션을 가로지르는 쿼리는 파티셔닝 없는 테이블보다 느릴 수 있다

---

## 10.4 Nullable 컬럼 과다 사용

### 왜 문제인가

Nullable 컬럼은 내부적으로 **별도의 null 마스크 컬럼**을 추가 유지한다. 이는:

- 스토리지 오버헤드 증가
- 쿼리 시 추가적인 null 체크 로직
- 압축 효율 저하

### 대안

- 빈 값과 null을 구분할 필요가 없으면, **기본값(0, 빈 문자열 등)**으로 대체
- 공식 best-practices: "Only use Nullable if explicitly required to distinguish between empty and null states."

---

## 10.5 데이터 타입 미최적화

| 안티패턴 | 올바른 선택 |
|---------|-----------|
| 숫자를 `String`으로 저장 | 적절한 `UInt`/`Int`/`Float` 타입 |
| 날짜를 `String`으로 저장 | `Date`, `Date32`, `DateTime` |
| 불필요하게 `DateTime64` 사용 | 밀리초 정밀도 필요 없으면 `DateTime` |
| `Int64` for 0~255 범위 | `UInt8` (8배 공간 절약) |
| 고유값 수천 이하의 문자열 | `LowCardinality(String)` (딕셔너리 인코딩) |
| 유한한 값 집합 | `Enum8` / `Enum16` (검증 + 효율) |

`LowCardinality`는 특히 효과적이다. 고유값 약 10,000개 이하의 컬럼에 적용하면 스토리지와 쿼리 성능이 극적으로 개선된다.

---

## 10.6 JOIN 과다 사용

ClickHouse에서 JOIN은 가능하지만, 단일 비정규화 테이블 쿼리보다 본질적으로 비용이 크다.

**권장**: 가능하면 **비정규화(denormalization)**로 JOIN을 제거하라. 데이터 중복이 발생하지만, 쿼리 레이턴시가 극적으로 감소한다.

JOIN이 불가피할 때:
- **최신 버전**(24.12+)을 사용하라. JOIN 성능이 매 릴리즈 개선 중
- 작은 테이블을 오른쪽에 배치 (24.12+에서는 자동 최적화)
- Dictionary 또는 IN 서브쿼리로 대체 가능한지 검토

---

## 10.7 Data Skipping Index 오용

Skipping index는 프라이머리 인덱스를 **대체하는 것이 아니라 보조**하는 수단이다.

순서를 지켜라:
1. 먼저 **ORDER BY 키를 최적화**하라 (7장)
2. 다음으로 **Materialized View**를 검토하라
3. 그래도 부족하면 **Skipping Index**를 추가

Skipping index의 효과는 **데이터 분포에 크게 의존**한다. 값이 granule 간에 고르게 분산되면 skip 효과가 없다. 추가 후 반드시 `EXPLAIN indexes = 1`로 실제 skip 비율을 확인하라.

---

---

# 11장. 모니터링과 트러블슈팅

---

## 11.1 핵심 시스템 테이블

ClickHouse의 모든 내부 상태는 `system.*` 테이블에서 관찰할 수 있다.

| 테이블 | 용도 |
|--------|------|
| `system.parts` | 파트 현황 (활성/비활성, 크기, 레벨, 파티션) |
| `system.merges` | 진행 중인 머지 (진행률, 대상 파트, 소요 시간) |
| `system.mutations` | 진행 중인 뮤테이션 (완료 여부, 영향 파트) |
| `system.replicas` | 레플리카 상태 (리더 여부, 지연, 큐 크기) |
| `system.query_log` | 쿼리 이력 (성능, 메모리, 읽기량) |
| `system.part_log` | 파트 이벤트 (생성, 머지, 삭제) |
| `system.metrics` | 실시간 메트릭 (스레드, 커넥션, 메모리) |
| `system.events` | 누적 이벤트 카운터 |
| `system.asynchronous_metrics` | 비동기 수집 메트릭 |

---

## 11.2 일상 모니터링 쿼리

### 파트 수 모니터링 (Too many parts 조기 감지)

```sql
SELECT
    database,
    `table`,
    count() AS active_parts,
    countIf(level = 0) AS unmerged_parts,
    formatReadableSize(sum(bytes_on_disk)) AS total_size
FROM system.parts
WHERE active
GROUP BY database, `table`
HAVING active_parts > 100
ORDER BY active_parts DESC;
```

`unmerged_parts`(level=0)가 지속 증가하면 머지가 따라잡지 못하는 상태다.

### 머지 진행 상황

```sql
SELECT
    database,
    `table`,
    round(progress * 100, 1) AS pct,
    num_parts,
    formatReadableSize(total_size_bytes_compressed) AS size,
    formatReadableTimeDelta(elapsed) AS elapsed
FROM system.merges
ORDER BY elapsed DESC;
```

### 뮤테이션 상태

```sql
SELECT
    database,
    `table`,
    mutation_id,
    command,
    is_done,
    parts_to_do,
    create_time
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time;
```

완료되지 않은 뮤테이션이 장시간 남아 있으면 머지 스레드를 점유한다.

### 레플리카 지연

```sql
SELECT
    database,
    `table`,
    replica_name,
    is_leader,
    absolute_delay,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    last_queue_update
FROM system.replicas
WHERE absolute_delay > 10
ORDER BY absolute_delay DESC;
```

### 슬로우 쿼리 식별

```sql
SELECT
    query_id,
    user,
    query_duration_ms,
    read_rows,
    formatReadableSize(read_bytes) AS read_size,
    formatReadableSize(memory_usage) AS memory,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
  AND query_duration_ms > 5000
ORDER BY query_duration_ms DESC
LIMIT 20;
```

### 현재 리소스 상태

```sql
SELECT metric, value, description
FROM system.metrics
WHERE metric IN (
    'Query', 'Merge', 'PartMutation',
    'GlobalThread', 'GlobalThreadActive',
    'MemoryTracking',
    'ReplicatedChecks', 'ReplicatedFetch'
)
ORDER BY metric;
```

---

## 11.3 트러블슈팅 가이드

### "Too many parts" 에러

**증상**: INSERT가 `DB::Exception: Too many parts` 에러로 거부됨

**원인 진단 순서**:

1. **소량 빈번 INSERT?** → 배칭 증가 또는 async insert 활성화
2. **과도한 파티셔닝?** → 파티션당 파트 수 확인. 파티션 키 카디널리티 검토
3. **머지 지연?** → `system.merges` 확인. `background_pool_size` 증가 고려
4. **뮤테이션 진행 중?** → `system.mutations` 확인. 미완료 뮤테이션이 머지 스레드 점유
5. **디스크 I/O 병목?** → `iostat`, `system.asynchronous_metrics`의 디스크 관련 메트릭 확인

### 레플리카 동기화 실패

**증상**: `system.replicas`에서 `absolute_delay` 지속 증가

**진단**:
```sql
-- 큐에 쌓인 작업 확인
SELECT
    type, count() AS cnt
FROM system.replication_queue
WHERE database = 'mydb' AND `table` = 'mytable'
GROUP BY type;
```

**주요 원인**: Keeper 연결 불안정, 네트워크 대역폭 부족, 원본 파트 누락, 디스크 공간 부족

### 쿼리 메모리 초과

**증상**: `DB::Exception: Memory limit exceeded`

**대응**:
```sql
-- 메모리 사용량이 큰 쿼리 식별
SELECT query_id, peak_memory_usage, query
FROM system.query_log
WHERE type = 'ExceptionWhileProcessing'
  AND exception LIKE '%Memory limit%'
ORDER BY event_time DESC
LIMIT 10;
```

흔한 원인: 고카디널리티 GROUP BY, 대용량 JOIN, ORDER BY 전체 결과셋 정렬

**대안**: `max_memory_usage` 설정 조정, `max_bytes_before_external_group_by`로 디스크 스필 활성화, 쿼리 리팩터링

### 머지가 느릴 때

```sql
-- 머지 처리량 추이
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS merges,
    avg(duration_ms) AS avg_ms,
    formatReadableSize(sum(written_bytes)) AS total_written
FROM system.part_log
WHERE event_type = 'MergeParts'
  AND event_date >= today() - 1
GROUP BY hour
ORDER BY hour;
```

처리량이 감소 추세면 디스크 성능 확인. SSD인지 HDD인지, IOPS 제한에 걸리는지 점검.

---

## 11.4 설정 변경 적용 확인

2장에서 다룬 대로, 설정 파일 에러 시 이전 설정이 유지된다. 변경 후 반드시 확인:

```sql
-- 현재 적용된 설정 값 확인
SELECT name, value, changed
FROM system.settings
WHERE name = 'max_threads';

-- 서버 설정 확인
SELECT name, value
FROM system.server_settings
WHERE name LIKE '%background%pool%';
```

---

## 11.5 Key Takeaways

| 항목 | 핵심 |
|------|------|
| **핵심 시스템 테이블** | parts, merges, mutations, replicas, query_log |
| **일상 모니터링** | 파트 수 추이, 머지 진행, 뮤테이션 상태, 레플리카 지연 |
| **Too many parts** | 배칭→파티셔닝→머지 스레드→뮤테이션→디스크 순서로 진단 |
| **레플리카 동기화** | replication_queue 확인. Keeper 연결 + 네트워크 + 디스크 점검 |
| **메모리 초과** | 고카디널리티 GROUP BY/JOIN이 주범. 디스크 스필 활성화 |
| **설정 확인** | system.settings, system.server_settings로 실제 적용값 검증 |

---

*이것으로 ClickHouse 학습 가이드의 전 11장을 완료합니다.*

---

# 부록: 참고 자료

| 자료 | 설명 |
|------|------|
| [Architecture Overview](/development/architecture) | 공식 내부 아키텍처 문서. 이 가이드의 2~3장의 원본 |
| [VLDB 2024 논문](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf) | ClickHouse의 학술적 아키텍처 개요 |
| [Why is ClickHouse so fast?](/concepts/why-clickhouse-is-so-fast) | 스토리지/쿼리 레이어 설계 철학 |
| [Core Concepts: Parts, Partitions, Merges, Primary Indexes, Shards](/parts) | 공식 Core Concepts 시리즈 |
| [Best Practices](/best-practices) | Primary Key, Partitioning Key, Insert Strategy, Data Types, Mutations, Joins, Skipping Indexes |
| [Sparse Primary Indexes Deep Dive](/guides/best-practices/sparse-primary-indexes) | 프라이머리 인덱스 심화 |
| [ClickHouse Keeper](/clickhouse/keeper) | ZooKeeper 대체 네이티브 솔루션 |
