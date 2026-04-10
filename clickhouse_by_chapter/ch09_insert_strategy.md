# 9장. INSERT 전략 선택

효율적인 데이터 수집(ingestion)은 ClickHouse의 고성능 운영의 기반이다. 잘못된 INSERT 전략은 "Too many parts" 에러, 머지 지연, 쿼리 성능 저하로 직결된다. 이 장에서는 공식 best-practices 문서를 기반으로 동기/비동기 INSERT, 배칭, 포맷, 압축, 멱등성을 다룬다.

---

## 9.1 동기 INSERT (기본 모드)

기본적으로 ClickHouse의 INSERT는 동기적이다. 각 INSERT 쿼리가 즉시 디스크에 파트를 생성하며, 메타데이터와 인덱스를 포함한다.

### INSERT의 서버 측 처리 흐름

```
클라이언트
  ① 배칭 (client-side)
  ② 사전 정렬 (optional)
  ③ 포맷 선택
  ④ 압축 (optional)
  ⑤ 네트워크 전송 (HTTP or Native)
        ↓
서버
  ⑥ 수신
  ⑦ 압축 해제
  ⑧ 포맷 파싱
  ⑨ MergeTree 포맷의 in-memory Block 구성
  ⑩ 프라이머리 키 순서로 정렬 (사전 정렬 시 skip)
  ⑪ Sparse primary index 생성
  ⑫ 컬럼별 압축
  ⑬ 디스크에 새 파트로 기록
```

### 배칭: 가장 중요한 최적화

INSERT에는 크기와 무관하게 **일정한 오버헤드**가 존재한다 (파트 생성, 인덱스 생성, 메타데이터 기록). 따라서 배치 크기가 클수록 오버헤드 비율이 줄어든다.

공식 권장:

> "We recommend inserting data in batches of at least 1,000 rows, and ideally between 10,000–100,000 rows."

그리고 INSERT 빈도에 대해:

> "We recommend keeping the number of insert queries around one insert query per second."

이유: INSERT마다 파트가 생성되고, 백그라운드 머지가 이를 따라잡아야 한다. 초당 수십~수백 회 INSERT하면 머지가 밀린다.

### 사전 정렬

클라이언트에서 프라이머리 키 순서로 데이터를 정렬하여 보내면, 서버가 ⑩ 정렬 단계를 skip할 수 있다. CPU 사용이 줄고 INSERT 속도가 향상된다.

**단, 선택적 최적화이다.** ClickHouse의 서버 측 정렬이 매우 효율적이므로(병렬 처리), 사전 정렬의 이점은 데이터가 이미 거의 정렬되어 있거나 클라이언트 자원이 충분할 때만 유의미하다. 고속 실시간 스트리밍(observability 등)에서는 사전 정렬을 건너뛰는 것이 일반적이다.

### INSERT의 멱등성 (Idempotency)

MergeTree 엔진은 기본적으로 **INSERT 중복 제거(deduplication)**를 수행한다. 동일한 배치 내용과 순서로 재시도하면 중복 파트가 생성되지 않는다.

보호하는 시나리오:
- INSERT는 성공했으나 네트워크 중단으로 클라이언트가 확인을 못 받은 경우
- 서버 측 INSERT 실패 후 타임아웃

**재시도 시 배치 내용과 순서를 동일하게 유지하는 것이 필수**이다.

---

## 9.2 비동기 INSERT (Async Insert)

### 언제 사용하는가

클라이언트 측 배칭이 불가능할 때 — 특히 observability 워크로드에서 수백~수천 에이전트가 실시간으로 소량 데이터를 지속 전송하는 환경.

### 동작 원리

```
에이전트 A ─┐
에이전트 B ─┤→ 서버 메모리 버퍼 ──[flush 조건 충족]──→ 하나의 파트 생성
에이전트 C ─┘

Flush 조건:
  (1) 버퍼 크기 초과 (async_insert_max_data_size)
  (2) 타임아웃 경과 (async_insert_busy_timeout_ms)
  (3) 누적 INSERT 쿼리 수 초과 (async_insert_max_query_number)
```

클라이언트에는 투명하게 동작한다. **Flush 전까지 데이터는 쿼리할 수 없다.**

### 반환 모드 선택

| 설정 | 동작 | 권장 여부 |
|------|------|-----------|
| `wait_for_async_insert = 1` (기본) | flush 완료 후 응답. 에러 발생 시 클라이언트에 전달 | **프로덕션 권장** |
| `wait_for_async_insert = 0` | 버퍼링 즉시 응답. fire-and-forget | 데이터 유실 허용 시에만 |

공식 문서의 강한 권고:

> "Using wait_for_async_insert=0 is very risky because your INSERT client may not be aware if there are errors, and also can cause potential overload."

### 활성화

```sql
-- 사용자 레벨
ALTER USER default SETTINGS async_insert = 1;

-- 쿼리 레벨
INSERT INTO mytable SETTINGS async_insert = 1 VALUES (...);
```

### 비동기 INSERT의 중복 제거

기본적으로 비동기 INSERT에서는 중복 제거가 **비활성화**되어 있다. 활성화할 수 있지만, 의존 Materialized View가 있으면 문제가 발생할 수 있으므로 주의가 필요하다.

---

## 9.3 포맷과 압축 선택

### 포맷 선택

| 포맷 | 특징 | 권장 용도 |
|------|------|-----------|
| **Native** | 가장 효율적. 컬럼 지향, 서버 측 파싱 최소. Go/Python 클라이언트 기본값 | 고처리량 수집 |
| **RowBinary** | 효율적인 행 기반 포맷. Java 클라이언트 기본값 | 클라이언트에서 컬럼 변환이 어려울 때 |
| **JSONEachRow** | 사용 편리하나 파싱 비용 높음 | 저볼륨, 빠른 통합 |

### 압축

**LZ4** (기본): 빠르고 가벼움. 50% 이상 데이터 크기 감소. 대부분의 경우 최적.

**ZSTD**: 높은 압축률이지만 CPU 부하 증가. 네트워크 전송 비용이 높을 때(리전 간, 클라우드 egress 비용).

Native 포맷 + LZ4 압축 조합이 최고 수집 성능을 달성한다.

---

## 9.4 인터페이스 선택: Native vs HTTP

| | Native 인터페이스 | HTTP 인터페이스 |
|---|-----------------|----------------|
| **포맷** | 항상 Native (최효율) | 70+ 포맷 지원 |
| **압축** | 블록 단위 LZ4/ZSTD | 전체 페이로드 단위 |
| **클라이언트 최적화** | MATERIALIZED/DEFAULT 컬럼 클라이언트 계산 가능 | 불가 |
| **로드밸런싱** | 어려움 | 용이 |
| **성능** | 약간 더 빠름 | 약간 느림 |
| **권장** | 최대 처리량이 필요할 때 | 대부분의 프로덕션 환경 |

공식 문서는 HTTP가 로드밸런서와 호환이 쉬워 **대부분의 경우 HTTP를 권장**한다. 성능 차이는 "small"이다.

### INSERT 대상 선택 (분산 환경)

- **로컬 테이블(ReplicatedMergeTree)에 직접 INSERT**: 가장 효율적. 클라이언트가 shard별 로드밸런싱
- **Distributed 테이블에 INSERT**: 간편하나 Initiator를 거치는 오버헤드. Initiator 장애 시 임시 데이터 유실 위험

---

## 9.5 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **배칭** | 최소 1,000행, 이상적으로 10,000~100,000행. 초당 약 1회 INSERT |
| **동기 INSERT** | 기본 모드. 배칭 가능하면 항상 사용. 멱등한 재시도 지원 |
| **비동기 INSERT** | 클라이언트 배칭 불가 시 대안. `wait_for_async_insert=1` 필수 |
| **포맷** | Native > RowBinary > JSONEachRow 순 효율 |
| **압축** | LZ4 기본. 네트워크 비용 높으면 ZSTD |
| **사전 정렬** | 선택적 최적화. 데이터가 이미 거의 정렬일 때 유의미 |
| **INSERT 대상** | 로컬 테이블 직접 INSERT가 가장 안전하고 효율적 |

---

*다음 장: 10장. 피해야 할 패턴들*
