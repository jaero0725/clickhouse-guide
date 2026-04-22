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

---

## 6.13 시각적 이해 — 고객 행동 데이터로 보는 머지 전 과정

### 테이블 설정

```sql
CREATE TABLE user_events (
    event_time  DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),
    page        LowCardinality(String),
    amount      UInt32
)
ENGINE = MergeTree
ORDER BY (event_type, toDate(event_time), user_id);
```

### Step 1 — INSERT 3번 → 파트 3개 생성

```
오전 9:00 INSERT 500만 건 →  all_0_0_0  (level 0)
오전 9:01 INSERT 500만 건 →  all_1_1_0  (level 0)
오전 9:02 INSERT 500만 건 →  all_2_2_0  (level 0)

디스크:
  user_events/
    all_0_0_0/
      event_type.bin, user_id.bin, amount.bin
      primary.idx, *.mrk
    all_1_1_0/
      event_type.bin, user_id.bin, amount.bin
      primary.idx, *.mrk
    all_2_2_0/
      ...
```

이 시점에 SELECT 하면 파트 3개를 모두 열어서 읽는다.

### Step 2 — 백그라운드 머지 4단계

```
① 압축 해제 & 로드
   all_0_0_0, all_1_1_0, all_2_2_0 의 .bin 파일을 메모리에 올림

② Merge Sort (정렬 순서 유지하며 합침)

   Part A: [click/2024-01-01/user_111, click/2024-01-01/user_999, ...]
   Part B: [click/2024-01-01/user_333, view/2024-01-01/user_222, ...]
   Part C: [click/2024-01-01/user_555, purchase/2024-01-02/user_777, ...]
                               ↓
   Merged: [click/2024-01-01/user_111,
            click/2024-01-01/user_333,
            click/2024-01-01/user_555,
            click/2024-01-01/user_999,
            purchase/2024-01-02/user_777,
            view/2024-01-01/user_222, ...]

③ 새 sparse primary index 생성 (8,192행마다 1개 엔트리)

④ 압축 & 저장 → all_0_2_1 (level=1) 생성
```

### Step 3 — 원본 파트 정리

```
all_0_2_1 생성 완료
  → all_0_0_0, all_1_1_0, all_2_2_0 → inactive 마킹
  → 8분 후 디스크 삭제 (old_parts_lifetime 기본값)

최종 디스크:
  user_events/
    all_0_2_1/   ← 3개가 1개로 수렴 (level 1)
      event_type.bin, user_id.bin, amount.bin
      primary.idx, *.mrk
```

### 시간에 따른 레벨 계층 형성

```
[오전 9시 ~ 12시 INSERT 계속]

  level 0 파트 수십 개  ←─ INSERT 직후
       ↓ 백그라운드 머지
  level 1 파트 몇 개
       ↓
  level 2 파트 1~2개
       ↓
  all_0_999_3  ← 결국 1개로 수렴 (파티셔닝 없는 경우)

system.parts 실제 스냅샷 예:
  all_0_5_1     level=1  rows=6,368,414   ← 최근 INSERT, 머지 진행 중
  all_6_11_1    level=1  rows=6,442,494
  all_0_23_2    level=2  rows=25,248,433  ← 오래된 데이터, 이미 수렴
```

### 엔진별 머지 시 다른 점

```
MergeTree (기본): 정렬 순서 유지하며 그냥 합침
  user_123 click  9:00  ← Part A
  user_123 click  9:01  ← Part B
            ↓ merge
  user_123 click  9:00  (둘 다 유지, 중복 허용)
  user_123 click  9:01

ReplacingMergeTree: 같은 키 중 최신만 유지
  user_123 grade=Bronze ver=1  ← Part A
  user_123 grade=Gold   ver=2  ← Part B
            ↓ merge
  user_123 grade=Gold   ver=2  (최신 ver만 살아남음)

SummingMergeTree: 같은 키의 숫자 컬럼 합산
  user_123 purchase_count=3 amount=15000  ← Part A
  user_123 purchase_count=2 amount= 8000  ← Part B
            ↓ merge
  user_123 purchase_count=5 amount=23000
```

---

## 6.14 시각적 이해 — FINAL 키워드가 느린 이유

### 문제 상황

```sql
CREATE TABLE user_grade (
    user_id UInt64,
    grade   String,
    ver     UInt64
)
ENGINE = ReplacingMergeTree(ver)
ORDER BY user_id;
```

```
오전: INSERT user_123 grade=Bronze ver=1  →  파트 A
낮:   INSERT user_123 grade=Gold   ver=2  →  파트 B
백그라운드 머지 아직 안 됨
```

```sql
-- 머지 전 SELECT 결과: 중복 2개
SELECT * FROM user_grade WHERE user_id = 123;
  user_123  Bronze  ver=1  ← 파트 A
  user_123  Gold    ver=2  ← 파트 B  (올바른 최신값)

-- FINAL 사용 시: 올바른 결과
SELECT * FROM user_grade FINAL WHERE user_id = 123;
  user_123  Gold    ver=2
```

### FINAL이 느린 이유 3가지

**이유 1 — 병렬 처리가 깨진다**

```
FINAL 없이:
  파트 A → 스레드 1  ┐
  파트 B → 스레드 2  ├── 완전 병렬, 각자 읽음
  파트 C → 스레드 3  ┘

FINAL 있으면:
  파트 A, B, C 전부 읽은 후
  같은 user_id끼리 ver 비교/정렬 → 단일 스레드 구간 발생
```

**이유 2 — 더 많은 데이터를 읽는다**

```
WHERE user_id = 123 이어도:
  FINAL 없이: primary index로 해당 granule만 읽음
  FINAL 있으면: user_id=123이 어느 파트에 있는지
               전체 파트를 더 넓게 스캔해야 함
```

**이유 3 — 메모리에서 머지 로직 실행 = 추가 CPU + 메모리**

```
파트가 100개 → 100개 읽고 메모리에서 정렬/비교
파트가 1개   → 이미 머지 완료 → FINAL이 할 일 없음

∴ 파트 수가 많을수록 FINAL 성능 저하 심각
```

### FINAL 대신 쓰는 패턴

```sql
-- argMax: ver이 최대인 행의 grade 반환
SELECT
    user_id,
    argMax(grade, ver) AS latest_grade
FROM user_grade
WHERE user_id = 123
GROUP BY user_id;

-- 병렬 실행 유지됨, FINAL보다 빠름
```
