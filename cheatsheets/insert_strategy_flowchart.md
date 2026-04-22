# 치트시트: INSERT 전략 의사결정 트리

"어떻게 넣어야 하지?"에 대한 빠른 답.

---

## 빠른 결정 플로우차트

```
데이터 INSERT 필요
  │
  ├── 클라이언트에서 배칭 가능한가?
  │     (예: Kafka consumer, 배치 job)
  │     │
  │     ├── YES → 동기 INSERT + 배치 (1만~10만 행)
  │     │          INSERT INTO ... VALUES (...), (...), ...
  │     │
  │     └── NO → Async Insert
  │                (ALTER USER SETTINGS async_insert=1)
  │
  ├── 초당 처리량이 얼마나 되는가?
  │     │
  │     ├── < 100 qps → 동기 INSERT OK
  │     ├── 100 ~ 10,000 qps → Async Insert 필수
  │     └── > 10,000 qps → 클라이언트 배칭 + 분산 샤딩
  │
  ├── 데이터 유실 허용?
  │     │
  │     ├── NO (금융, 주문 등) → wait_for_async_insert=1
  │     │                         insert_quorum > 1 (분산)
  │     │
  │     └── YES (로그, 메트릭) → 기본 설정
  │
  └── 분산 환경인가?
        │
        ├── YES → Distributed 테이블에 INSERT 대신
        │          각 shard의 로컬 테이블에 직접 INSERT
        │          (클라이언트 또는 LB에서 라우팅)
        │
        └── NO → 일반 INSERT
```

---

## 시나리오별 추천 설정

### 시나리오 1: Kafka/ETL 배치 파이프라인

```
특징: 클라이언트에서 배칭 가능, 대용량
추천: 동기 INSERT + Native 포맷 + LZ4
```

```python
# 권장 배치 크기: 10,000 ~ 100,000 행
buffer = []
for message in consumer:
    buffer.append(message)
    if len(buffer) >= 50000:
        client.insert(
            'events', buffer,
            column_names=[...],
            settings={'input_format_defaults_for_omitted_fields': 1}
        )
        buffer = []
```

**왜 이 설정?**
- Native 포맷: 파싱 오버헤드 최소 (2~3배 빠름)
- LZ4: 적당한 압축 + 빠른 속도
- 동기 INSERT: 가장 효율적 경로

---

### 시나리오 2: 수백 개 앱 서버에서 로그 수집

```
특징: 배칭 불가, 수많은 클라이언트, 고속
추천: Async Insert
```

```sql
-- 유저 단위 설정
CREATE USER app_user IDENTIFIED BY '...';
ALTER USER app_user SETTINGS
    async_insert = 1,
    wait_for_async_insert = 1,              -- 프로덕션 권장
    async_insert_max_data_size = 10000000,  -- 10MB 버퍼
    async_insert_busy_timeout_ms = 1000,    -- 1초 타임아웃
    async_insert_max_query_number = 450;    -- 450개 쿼리 모이면 flush
```

```
동작:
  App1, App2, ..., AppN → 각자 INSERT
         ↓
  서버에서 메모리 버퍼링
         ↓
  flush 조건 (크기 or 시간 or 쿼리 수) 충족
         ↓
  하나의 파트로 생성
```

---

### 시나리오 3: IoT 센서 (수만 기기, 느린 저전력 디바이스)

```
특징: 각 디바이스가 소량 데이터, 불안정 네트워크
추천: Async Insert + HTTP 인터페이스
```

```
디바이스 → HTTP POST → LB → ClickHouse
                              │
                        async_insert=1
                        wait_for_async_insert=0 (유실 허용)
```

wait_for_async_insert=0 이 위험한 이유 알면서도 여기서 쓰는 이유:
- 센서 데이터 1건 유실은 큰 문제 아님
- 디바이스가 에러 처리할 수 없음
- 응답 시간 최소화 (배터리 절약)

---

### 시나리오 4: 트랜잭셔널 (주문, 결제)

```
특징: 데이터 유실 절대 불가, 중복도 금지
추천: 동기 INSERT + 멱등성 활용 + Quorum
```

```sql
-- 분산 환경에서
SET insert_quorum = 2;              -- 2개 레플리카에 확인 후 완료
SET insert_quorum_timeout = 60000;  -- 1분 대기

INSERT INTO orders VALUES (...);
-- 네트워크 장애 시 → 동일한 배치로 재시도 (deduplication 자동)
```

**멱등성 보장:**
```
MergeTree는 같은 배치(내용+순서)를 재시도하면
자동으로 중복 제거 (파트 hash 비교)

→ "INSERT 성공했나?" 모를 때 그냥 재시도하면 안전
```

---

### 시나리오 5: 대용량 일회성 Backfill

```
특징: 과거 데이터 수TB 일괄 투입
추천: Sorted batch + Native format + 큰 max_insert_block_size
```

```sql
-- 세션 설정으로 한 번에 큰 블록 처리
SET max_insert_block_size = 1048576;  -- 100만 행
SET input_format_parallel_parsing = 1;

-- 사전 정렬된 데이터 투입 (서버 정렬 단계 skip)
INSERT INTO mytable FORMAT Native ...;
```

```
참고: s3 함수로 직접 읽기
INSERT INTO mytable
SELECT * FROM s3('s3://bucket/data/*.parquet', 'Parquet');
```

---

## Async Insert 세부 설정

```sql
-- 핵심 설정 5개
async_insert                       = 1      -- 활성화
wait_for_async_insert              = 1      -- 응답 대기 (권장)
async_insert_max_data_size         = 10MB   -- 버퍼 크기 한계
async_insert_busy_timeout_ms       = 1000   -- flush 타임아웃
async_insert_max_query_number      = 450    -- 누적 쿼리 수 한계

-- 고급
async_insert_deduplicate           = 0      -- MV 의존 시 주의
async_insert_threads               = 16     -- 처리 스레드
async_insert_stale_timeout_ms      = 0      -- 유휴 타임아웃
```

### 권장 조합

| 환경 | wait_for | max_data_size | busy_timeout |
|------|----------|--------------|--------------|
| 프로덕션 (일반) | 1 | 10MB | 1000ms |
| 프로덕션 (고처리량) | 1 | 100MB | 200ms |
| 저전력 디바이스 | 0 | 1MB | 5000ms |
| 금융/주문 | async_insert 쓰지 말 것 |

---

## 포맷 선택

| 포맷 | 파싱 비용 | 호환성 | 용도 |
|------|---------|--------|------|
| **Native** | 가장 빠름 | CH 클라이언트만 | 최대 성능 |
| **RowBinary** | 빠름 | 커스텀 클라이언트 | Java 등 |
| **Parquet** | 중간 | 범용 | DW 백필 |
| **CSV** | 느림 | 보편적 | 임시/수동 |
| **JSONEachRow** | 가장 느림 | 유연 | 빠른 프로토타이핑 |

---

## 압축 선택

```
네트워크 상 데이터:
  일반 환경: LZ4 (속도 우선)
  네트워크 비용 높음: ZSTD (압축률 우선)
  
  LZ4:  압축률 ~50%, 속도 매우 빠름
  ZSTD: 압축률 ~70%, 속도 LZ4의 1/3

→ 같은 리전 내: LZ4
→ 리전 간 / S3: ZSTD
```

---

## INSERT 대상 선택 (분산 환경)

### Distributed 테이블에 INSERT (편하지만 위험)

```sql
INSERT INTO events_distributed VALUES ...;
```

```
동작:
  Initiator 서버가 데이터를 받음
  → 임시 파일로 저장
  → 백그라운드로 각 shard에 전송

위험:
  Initiator 장애 시 임시 파일 유실
```

### 로컬 테이블에 직접 INSERT (권장)

```
애플리케이션이 직접 라우팅:
  order_id % 4 == 0 → shard-01의 events_local
  order_id % 4 == 1 → shard-02의 events_local
  ...

장점:
  Initiator 중계 없음
  각 shard에 직접 도달
  장애 내성 향상
```

---

## 체크리스트: 새 INSERT 파이프라인 만들 때

- [ ] 배칭 크기 1만 행 이상인가?
- [ ] 클라이언트에서 배칭 불가하면 Async Insert 설정했는가?
- [ ] `wait_for_async_insert = 1` (데이터 유실 허용 안 됨)?
- [ ] 포맷은 Native인가? (최대 성능 필요 시)
- [ ] 압축 활성화했는가?
- [ ] 분산 환경에서 로컬 테이블 직접 INSERT인가?
- [ ] 멱등한 재시도 로직 있는가?
- [ ] Partition 키 카디널리티 10,000 이하인가?
- [ ] INSERT 빈도 초당 1회 수준으로 수렴하는가?
- [ ] 모니터링 설정: `system.asynchronous_insert_log`, 파트 수, "Too many parts" 알림
