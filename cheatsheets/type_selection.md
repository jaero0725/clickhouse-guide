# 치트시트: 데이터 타입 선택 가이드

ClickHouse는 컬럼 지향이라 **타입 크기가 직접 성능에 영향**을 준다. Int64로 통일하는 습관은 2배 비용을 발생시킨다.

---

## 정수 타입

| 타입 | 범위 | 크기 | 용도 |
|------|------|------|------|
| `Int8` | -128 ~ 127 | 1B | 플래그, 작은 상태값 |
| `Int16` | -32768 ~ 32767 | 2B | 작은 정수 |
| `Int32` | ±21억 | 4B | 일반 정수, auto increment id (작은 서비스) |
| `Int64` | 매우 큼 | 8B | 큰 id, 타임스탬프 (microsecond) |
| `Int128`, `Int256` | 극대 | 16B/32B | 특수 용도 |

**Unsigned 버전** (`UInt8` 등)은 음수가 없을 때 범위 2배 활용 가능.

### 선택 규칙

```
금액 (원): 42억 이상 가능? → UInt64, 아니면 UInt32
유저 ID: 40억 유저? → UInt64, 작은 서비스면 UInt32
카운터 (조회수 등): UInt32 (42억)
페이지 크기, 순번: UInt16 (6.5만)
플래그, 등급: UInt8 (255)
```

---

## 실수 타입

| 타입 | 정밀도 | 크기 | 용도 |
|------|--------|------|------|
| `Float32` | 7자리 | 4B | 센서값, 좌표 |
| `Float64` | 15자리 | 8B | 과학 계산 |
| `Decimal(P, S)` | 정확 | 가변 | **금액, 환율** (반올림 오차 없음) |

**금액에 Float 쓰지 말 것:**
```
Float으로 누적:
  0.1 + 0.2 = 0.30000000000000004  ← 반올림 오차
  
Decimal 또는 정수(센트/원 단위)로:
  10 + 20 = 30  ← 정확
```

---

## 문자열 타입

| 타입 | 용도 |
|------|------|
| `String` | 일반 텍스트 |
| `FixedString(N)` | 고정 길이 (ipv4는 4, uuid 문자열은 36 등) |
| `LowCardinality(String)` | **고유값 수천 이하인 문자열** |
| `UUID` | UUID (16바이트 내부 저장) |

### LowCardinality가 핵심

```sql
-- 비효율 (압축해도 여전히 느림)
country String             -- 'Korea', 'USA', 'Japan', ... 수십 개

-- 효율 (딕셔너리 인코딩)
country LowCardinality(String)
```

**내부 동작:**
```
딕셔너리: {0: 'Korea', 1: 'USA', 2: 'Japan', ...}
컬럼 데이터: [0, 1, 0, 2, 0, 1, ...]   ← UInt8/16 저장

→ 문자열 비교가 정수 비교로 치환
→ 스토리지 5~10배 절감
→ GROUP BY/WHERE 속도 대폭 향상
```

**기준:**
- 고유값 ≤ 10,000: LowCardinality 강력 권장
- 고유값 ≤ 100,000: LowCardinality 권장
- 고유값 > 100,000: 일반 String이 나을 수 있음

---

## 날짜/시간 타입

| 타입 | 정밀도 | 크기 | 범위 |
|------|--------|------|------|
| `Date` | 일 | 2B | 1970 ~ 2149 |
| `Date32` | 일 | 4B | 1900 ~ 2299 |
| `DateTime` | 초 | 4B | 1970 ~ 2105 |
| `DateTime64(3)` | 밀리초 | 8B | 고정밀 |
| `DateTime64(6)` | 마이크로초 | 8B | 로그/트레이싱 |
| `DateTime64(9)` | 나노초 | 8B | 고정밀 과학 |

### 선택 규칙

```
생년월일, 날짜만 필요 → Date
주문 시각, 이벤트 시각 → DateTime (초)
로그, 트레이싱 → DateTime64(3) 또는 DateTime64(6)
```

**ORDER BY 키 설계 시:**
```sql
-- 나쁜 예 (DateTime64(6)가 8바이트)
ORDER BY (event_time_precise, user_id)

-- 좋은 예 (날짜만으로 인덱스 → 2바이트)
ORDER BY (toDate(event_time_precise), user_id)
```

---

## Array, Tuple, Map, Nested

### Array

```sql
tags Array(String)
scores Array(Float32)
```

동적 길이의 리스트. 배열 함수 풍부:
```sql
SELECT has(tags, 'sale'), arrayMap(x -> x * 2, scores);
```

### Tuple

```sql
point Tuple(Float32, Float32)    -- (x, y)
geo   Tuple(lat Float64, lon Float64)  -- 이름 있는 튜플
```

### Map

```sql
attrs Map(String, String)
```

런타임에 키가 변하는 데이터. **단, 필터링이 느림** → 자주 쓰는 키는 별도 컬럼으로 승격.

### Nested (사실상 병렬 Array)

```sql
items Nested (
    product_id UInt32,
    quantity UInt16,
    price UInt32
)
-- 내부: items.product_id Array(UInt32), items.quantity Array(UInt16), ...
```

---

## Nullable — 가능하면 피하기

```sql
-- 나쁜 예
age Nullable(UInt8)

-- 좋은 예
age UInt8 DEFAULT 0
```

**Nullable의 내부 동작:**
```
Nullable(UInt8) = UInt8 컬럼 + 별도 UInt8 마스크 컬럼
→ 저장 공간 2배
→ 모든 연산이 마스크 체크 → 성능 저하

거의 항상 DEFAULT 값으로 대체 가능
정말 NULL 의미가 필요할 때만 사용
```

---

## Enum

```sql
status Enum8('active' = 1, 'inactive' = 2, 'banned' = 3)
```

고정된 문자열 집합에 유리. 내부적으로 정수 저장.

**LowCardinality vs Enum:**

```
LowCardinality: 런타임에 새 값 추가 가능
Enum:           ALTER TABLE로만 값 추가, 더 빠름
```

작은 고정 집합(상태, 타입) → Enum
많은 값 + 확장 가능성 → LowCardinality

---

## 압축 코덱

```sql
CREATE TABLE t (
    ts    DateTime CODEC(DoubleDelta, LZ4),    -- 시계열 타임스탬프
    value Float32  CODEC(Gorilla, LZ4),         -- 시계열 실수
    text  String   CODEC(ZSTD(3)),              -- 긴 텍스트
    id    UInt64   CODEC(Delta, LZ4)            -- 증가하는 ID
);
```

### 코덱 선택

| 데이터 특성 | 권장 코덱 |
|-----------|---------|
| 기본 (특성 모름) | `LZ4` (기본값) |
| 고압축 필요 | `ZSTD(3)` ~ `ZSTD(9)` |
| 증가하는 타임스탬프 | `DoubleDelta` + `LZ4` |
| 단조 증가 정수 | `Delta` + `LZ4` |
| 실수 시계열 (센서) | `Gorilla` + `LZ4` |
| 고카디널리티 String | `ZSTD` |
| 저카디널리티 String | `LowCardinality`로 먼저 |

### 압축 효과 예시

```
DateTime 1억 행 (10초 간격):
  LZ4 기본:            410 MB
  DoubleDelta + LZ4:    12 MB   ← 34배 차이

Float32 1억 행 (센서값):
  LZ4:                 380 MB
  Gorilla + LZ4:        95 MB   ← 4배 차이
```

---

## 흔한 실수

| 실수 | 영향 | 해결 |
|------|------|------|
| 모든 정수 UInt64 | 공간 2~8배 낭비 | 최소 타입 선택 |
| 금액을 Float | 반올림 오차 | 정수(센트) 또는 Decimal |
| 고유값 적은 String | 압축률·속도 저하 | LowCardinality |
| Nullable 남용 | 공간 2배, 느림 | DEFAULT 값 |
| DateTime64(9) 습관적 | 8바이트, 인덱스 크기 | DateTime (4B) |
| 코덱 미지정 | 압축률 저조 | 데이터 특성별 코덱 |

---

## 실무 템플릿

### 이벤트 로그 테이블

```sql
CREATE TABLE events (
    ts          DateTime64(3) CODEC(DoubleDelta, LZ4),
    event_type  LowCardinality(String),
    user_id     UInt64,
    device      LowCardinality(String),
    country     LowCardinality(String),
    amount      UInt32,                    -- 센트/원 단위
    props       Map(String, String) CODEC(ZSTD(3))
);
```

### 시계열 메트릭 테이블

```sql
CREATE TABLE metrics (
    ts     DateTime CODEC(DoubleDelta, LZ4),
    device UInt32,
    value  Float32 CODEC(Gorilla, LZ4)
);
```

### 엔티티 상태 테이블

```sql
CREATE TABLE entity (
    id          UInt64,
    status      LowCardinality(String),
    updated_at  DateTime,
    version     UInt64
)
ENGINE = ReplacingMergeTree(version);
```
