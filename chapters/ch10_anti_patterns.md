# 10장. 피해야 할 패턴들

ClickHouse는 올바르게 사용하면 놀라운 성능을 보여주지만, OLTP 데이터베이스의 습관을 그대로 가져오면 심각한 성능 문제에 직면한다. 이 장에서는 공식 best-practices 문서에서 명시적으로 경고하는 안티패턴들을 정리한다.

---

## 10.1 Mutation 남용: ALTER TABLE UPDATE/DELETE

### 왜 위험한가

ClickHouse에서 **mutation**은 `ALTER TABLE ... DELETE`와 `ALTER TABLE ... UPDATE`를 의미한다. SQL 문법은 표준과 비슷하지만, 내부 동작은 근본적으로 다르다.

공식 문서:

> "Rather than modifying rows in place, mutations in ClickHouse are asynchronous background processes that rewrite entire data parts affected by the change."

MergeTree의 immutable 파트 모델 때문에, 단 하나의 행을 수정하더라도 **해당 행이 포함된 파트 전체를 다시 쓴다**. 이는 막대한 write amplification과 디스크 I/O를 발생시킨다.

### Mutation의 특성

- **비동기 실행**: 제출 즉시 완료되지 않음. 백그라운드에서 처리
- **롤백 불가**: 제출 후 취소만 가능 (`KILL MUTATION`). 되돌리기 없음
- **서버 재시작에도 지속**: 재시작 후에도 큐에 남아 계속 실행
- **SELECT와의 불일치**: mutation 진행 중 SELECT는 mutated + unmutated 파트의 혼합을 읽을 수 있음
- **Totally ordered**: mutation 제출 이전 데이터에만 적용. 이후 INSERT된 데이터는 영향 없음

### 대안

- **ReplacingMergeTree**: 최신 행만 유지 (6장 참조). UPDATE 대신 새 행 INSERT
- **CollapsingMergeTree**: sign 컬럼 기반 취소. DELETE 대신 sign=-1 행 INSERT
- **Lightweight Delete** (`DELETE FROM`): mutation보다 가벼운 삭제. 파트를 다시 쓰지 않고 마스크 파일로 처리
- **파티션 DROP**: 대량 삭제는 파티션 단위로 (5장 참조). 초 단위 완료

```sql
-- 피하라
ALTER TABLE mytable UPDATE status = 'inactive' WHERE user_id = 123;
ALTER TABLE mytable DELETE WHERE date < '2023-01-01';

-- 대안: ReplacingMergeTree에 새 행 INSERT
INSERT INTO mytable VALUES (123, 'inactive', now());

-- 대안: 파티션 단위 삭제
ALTER TABLE mytable DROP PARTITION '2022-01-01';
```

---

## 10.2 OPTIMIZE TABLE FINAL 남용

### 왜 위험한가

```sql
OPTIMIZE TABLE mytable FINAL;
```

이 명령은 **모든 활성 파트를 하나의 파트로 강제 머지**한다.

공식 문서의 경고:

> "You should avoid the OPTIMIZE FINAL operation in most cases as it initiates resource intensive operations which may impact cluster performance."

### 구체적 문제

**비용이 크다**: 모든 파트를 압축 해제 → 병합 → 재압축 → 디스크 기록. CPU와 I/O를 극심하게 소비한다.

**안전 제한을 무시한다**: 일반 백그라운드 머지는 ~150GB(`max_bytes_to_merge_at_max_space_in_pool`) 제한을 준수하지만, `OPTIMIZE FINAL`은 이 제한을 **무시**한다. 여러 150GB 파트를 하나로 합치려 시도할 수 있으며, 이는 메모리 부족, 장시간 머지, OOM으로 이어질 수 있다.

**역효과**: 생성된 거대 파트가 이후 머지에서 문제가 된다. ReplacingMergeTree에서는 중복이 누적되는 상황이 발생할 수 있다.

### OPTIMIZE FINAL ≠ FINAL

주의: 쿼리의 `FINAL` 키워드와 혼동하지 말 것.

```sql
-- OPTIMIZE FINAL: 물리적으로 파트를 병합. 피해야 함
OPTIMIZE TABLE mytable FINAL;

-- SELECT ... FINAL: 쿼리 시점에 머지 로직 적용. 필요하면 사용 가능
SELECT * FROM mytable FINAL WHERE ...;
```

`SELECT ... FINAL`은 프라이머리 키 필터가 있으면 성능이 괜찮을 수 있다.

### 대안

**백그라운드 머지에 맡겨라.** ClickHouse의 자동 머지가 파트 수와 크기를 최적으로 관리한다. 수동 개입은 거의 항상 불필요하고 유해하다.

---

## 10.3 과도한 파티셔닝

5장에서 상세히 다뤘으므로 핵심만 재강조한다:

- 파티션 수 1,000~10,000 이하 유지
- 파티션 간 머지 불가 → 파트 수 폭증
- 파티션 키가 프라이머리 키에 이미 있으면 중복 효과
- **쿼리 최적화 목적이라면 대부분 파티셔닝하지 않는 것이 낫다**

---

## 10.4 Nullable 컬럼 남용

```sql
-- 피하라
CREATE TABLE t (x Int8, y Nullable(Int8)) ENGINE = MergeTree ORDER BY x;

-- 대안
CREATE TABLE t (x Int8, y Int8 DEFAULT 0) ENGINE = MergeTree ORDER BY x;
```

`Nullable(T)` 타입은 내부적으로 **별도의 `UInt8` 컬럼**을 추가로 생성한다. 이 컬럼은 각 행이 NULL인지 여부를 추적한다.

결과:
- 스토리지 공간 증가
- 모든 Nullable 컬럼 연산에서 추가 컬럼 처리 필요
- **거의 항상 성능 저하**

NULL이 비즈니스 로직상 반드시 필요한 경우가 아니면, DEFAULT 값을 지정하여 Nullable을 회피하라.

---

## 10.5 소량 빈번 INSERT

4장과 9장에서 반복적으로 강조한 내용:

- INSERT마다 새 파트 생성 → 초당 수백~수천 INSERT는 파트 폭증
- `parts_to_throw_insert` (기본 3000) 초과 시 INSERT 거부
- **최소 1,000행, 이상적으로 10,000~100,000행 배치**
- 배칭 불가 시 async insert 활용

---

## 10.6 Distributed 테이블에 직접 INSERT

8장에서 다뤘듯이, Distributed 테이블 INSERT는 Initiator 서버에 임시 저장 후 비동기 전달된다. 위험:

- Initiator 장애 시 임시 데이터 유실
- 추가 네트워크 홉으로 레이턴시 증가
- shard 간 데이터 불균형 감지가 어려움

**권장**: 로컬 테이블(ReplicatedMergeTree)에 직접 INSERT. 클라이언트 또는 로드밸런서가 shard별 라우팅 담당.

---

## 10.7 부적절한 데이터 타입 선택

### LowCardinality 미사용

카디널리티가 낮은(수천 이하 고유값) String 컬럼에는 `LowCardinality(String)`을 사용하라. 내부적으로 딕셔너리 인코딩을 적용하여 스토리지와 쿼리 성능 모두 크게 향상된다.

```sql
-- 비효율
town String

-- 효율
town LowCardinality(String)
```

### 과도하게 넓은 타입

`Int64`가 필요 없는데 습관적으로 사용하지 마라. `UInt16`으로 충분하면 `UInt16`을 쓰라. 컬럼 지향 스토리지에서는 타입 크기가 직접적으로 디스크 사용량과 메모리 소비에 영향을 미친다.

### DateTime 정밀도

일(day) 단위로 충분하면 `Date`(2바이트)를 사용하라. `DateTime`(4바이트)이나 `DateTime64`(8바이트)는 필요할 때만.

---

## 10.8 Key Takeaways

| 안티패턴 | 문제 | 대안 |
|----------|------|------|
| **Mutation (UPDATE/DELETE)** | 파트 전체 재작성. I/O 폭증 | ReplacingMergeTree, Lightweight Delete, 파티션 DROP |
| **OPTIMIZE FINAL** | 모든 파트 강제 병합. 안전 제한 무시 | 백그라운드 머지에 위임 |
| **과도한 파티셔닝** | 파트 분산 → 머지 불가 → Too many parts | 저카디널리티 키, 1000 이하 파티션 |
| **Nullable 컬럼** | 추가 UInt8 컬럼 → 스토리지·성능 저하 | DEFAULT 값 사용 |
| **소량 빈번 INSERT** | 파트 폭증 → 머지 지연 → INSERT 거부 | 배칭, async insert |
| **Distributed INSERT** | Initiator 임시 저장 → 장애 시 유실 | 로컬 테이블 직접 INSERT |
| **부적절한 타입** | 불필요한 스토리지·메모리 소비 | LowCardinality, 적정 크기 타입 |

---

*다음 장: 11장. 모니터링과 트러블슈팅*
