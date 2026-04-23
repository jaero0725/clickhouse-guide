# D3. Mutation 남발 — 백그라운드 머지 마비 + 디스크 2배

## 증상

- `ALTER TABLE UPDATE/DELETE` 실행 후 테이블 쿼리가 점점 느려짐
- `system.mutations`에 is_done=0인 Mutation이 쌓임
- 디스크 사용량이 갑자기 2배로 증가
- 백그라운드 머지가 진행되지 않음

---

## 원인

ClickHouse의 `ALTER TABLE UPDATE/DELETE`는 MySQL의 UPDATE/DELETE와 전혀 다르다. 이것은 **Mutation**이라 불리며, 파트 전체를 다시 쓰는 작업이다.

```
MySQL UPDATE:
  해당 행의 값만 변경 (인플레이스 또는 버퍼 내)
  → 빠름, 영향 범위 최소

ClickHouse ALTER TABLE UPDATE:
  1. 조건에 해당하는 행이 있는 모든 파트 식별
  2. 각 파트를 전부 읽어 새 파트로 재기록
  3. 기존 파트 마킹 후 제거
  → 전체 파트 재기록 = 대규모 I/O 작업
  → 백그라운드에서 비동기로 실행
```

```
예시: 10GB 테이블, 1행만 UPDATE
  → 10GB 전체 읽기 → 수정 → 10GB 쓰기
  → 완료 전까지 기존 10GB + 새 10GB = 20GB 필요
  → Mutation 완료까지 머지가 일시 중단
```

**반복 Mutation이 쌓이면:**

```
Mutation 1 (진행 중): parts_to_do = 42, is_done = 0
Mutation 2 (대기):    parts_to_do = 42, is_done = 0
Mutation 3 (대기):    parts_to_do = 42, is_done = 0

→ 백그라운드 스레드가 Mutation에 묶여 일반 머지 불가
→ 파트 폭증 → "Too many parts"
```

---

## 진단

```sql
-- 현재 진행/대기 중인 Mutation 목록
SELECT
    table,
    mutation_id,
    command,
    parts_to_do,
    parts_done,
    is_done,
    create_time,
    latest_fail_reason
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time;

-- Mutation이 처리 중인 파트 수 vs 전체 파트 수
SELECT
    m.table,
    m.parts_to_do,
    p.total_parts,
    round(m.parts_done * 100.0 / (m.parts_to_do + m.parts_done), 1) AS pct_done
FROM system.mutations m
JOIN (
    SELECT table, count() AS total_parts FROM system.parts
    WHERE active GROUP BY table
) p ON m.table = p.table
WHERE NOT m.is_done;

-- 디스크 사용량 이상 확인 (Mutation 임시 파트)
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active = 0         -- 비활성 파트 (Mutation 대기 중인 구 파트)
GROUP BY table;
```

---

## 해결

### 즉각 조치 — Mutation 취소

```sql
-- 특정 Mutation ID 취소
KILL MUTATION WHERE mutation_id = 'mutation_4.txt';

-- 특정 테이블의 모든 Mutation 취소
KILL MUTATION WHERE database = 'mydb' AND table = 'events';

-- 취소 확인
SELECT * FROM system.mutations WHERE NOT is_done;
```

### 근본 해결 1 — INSERT-only 패턴으로 전환

**상태 변경이 필요한 경우 UPDATE 대신 새 행 INSERT.**

```sql
-- 나쁜 예: 주문 상태 변경
ALTER TABLE orders
UPDATE status = 'shipped'
WHERE order_id = 12345;   -- 파트 전체 재기록

-- 좋은 예: 이벤트 소싱 패턴
INSERT INTO orders VALUES
(12345, 'shipped', now(), ...);   -- 상태 변경 이벤트 추가

-- 최신 상태 조회
SELECT order_id, argMax(status, updated_at)
FROM orders WHERE order_id = 12345
GROUP BY order_id;
```

### 근본 해결 2 — ReplacingMergeTree 사용

```sql
-- 상태 변경이 잦은 테이블: ReplacingMergeTree
CREATE TABLE orders (
    order_id   UInt64,
    user_id    UInt64,
    status     LowCardinality(String),
    amount     UInt32,
    updated_at DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY order_id;

-- UPDATE 대신 최신 상태 INSERT
INSERT INTO orders VALUES (12345, 42, 'shipped', 5000, now());
-- 이전 행은 머지 시 자동 제거
```

### 근본 해결 3 — 불가피한 DELETE는 파티션 DROP으로

대량 데이터 삭제가 필요할 때.

```sql
-- 나쁜 예: 대량 DELETE
ALTER TABLE logs DELETE WHERE date < '2024-01-01';
-- → 수 TB 파트 전체 재기록

-- 좋은 예: 파티션 DROP (즉시 완료)
ALTER TABLE logs DROP PARTITION '2023-12';
ALTER TABLE logs DROP PARTITION '2023-11';
-- → 파티션 파일 삭제만 (밀리초)

-- 자동화: TTL
CREATE TABLE logs (
    ...
    date Date
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (...)
TTL date + INTERVAL 6 MONTH DELETE;  -- 6개월 후 자동 삭제
```

### 경량 DELETE (ClickHouse 23.3+)

작은 범위의 DELETE는 `DELETE FROM` 문법으로 경량 처리 가능.

```sql
-- 일반 Mutation (느림, 파트 재기록)
ALTER TABLE events DELETE WHERE user_id = 42;

-- 경량 DELETE (빠름, 삭제 마크만 기록)
DELETE FROM events WHERE user_id = 42;
-- 실제 파트 재기록은 다음 머지 시 처리
```

### 경량 UPDATE (ClickHouse 25.7+)

```sql
-- 표준 SQL UPDATE (경량, 패치 파트 방식)
UPDATE events SET status = 'archived' WHERE event_id = 9999;
-- 전체 파트 재기록 없이 패치 파트만 기록
```

---

## 예방

```
Mutation 사용 기준:

  ✅ 일회성 데이터 수정 (마이그레이션, 오류 정정)
  ✅ 소수 파트에 영향을 미치는 경우
  ✅ 트래픽 없는 시간대 실행
  ✅ 1개씩 실행 (여러 개 동시 금지)

  ❌ 주기적인 상태 업데이트 → INSERT-only 패턴
  ❌ 대량 삭제 → 파티션 DROP + TTL
  ❌ 고빈도 UPDATE → ReplacingMergeTree

  실행 전 체크:
  SELECT count() FROM system.mutations WHERE NOT is_done;
  → 0이 아니면 현재 Mutation 완료 후 실행
```
