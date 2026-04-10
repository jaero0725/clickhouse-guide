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
