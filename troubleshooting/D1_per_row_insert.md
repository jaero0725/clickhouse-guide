# D1. 건당 INSERT — "Too many parts" DB 즉사

## 증상

```
DB::Exception: Too many parts (3001).
Merges are processing significantly slower than inserts.
Consider slowing down inserts or increasing background_pool_size.
```

- 서비스 초기에는 정상이다가 트래픽 증가 후 갑자기 DB 응답 불가
- INSERT 응답 시간이 100ms → 수 초 → 타임아웃 순으로 악화
- `system.parts` 파트 수가 수천~수만 개

---

## 원인

ClickHouse에는 MySQL InnoDB 같은 **메모리 버퍼(MemTable)가 없다.**

```
MySQL InnoDB INSERT 흐름:
  INSERT → Buffer Pool (메모리) → 나중에 디스크 플러시

ClickHouse MergeTree INSERT 흐름:
  INSERT → 즉시 디스크에 새 Part 파일 생성
```

```
결과:
  건당 INSERT 100개/초 → 초당 파트 100개 생성
  백그라운드 머지: 초당 최대 약 10~20파트 처리

  1분 후: 파트 = 6,000 - 1,200 = 4,800개
  3,000개 초과 시점: "Too many parts" 에러로 INSERT 차단
```

---

## 진단

```sql
-- 파트 수 현황
SELECT table, count() AS parts
FROM system.parts
WHERE database = 'mydb' AND active
GROUP BY table
ORDER BY parts DESC;
-- 3,000 이상: 위험

-- 현재 머지 속도 vs INSERT 속도 비교
SELECT
    event_type,
    count() AS events
FROM system.part_log
WHERE event_time >= now() - INTERVAL 1 MINUTE
GROUP BY event_type;
-- NewPart: 초당 INSERT 횟수
-- MergeParts: 초당 머지 횟수 → NewPart >> MergeParts이면 폭증 중

-- 파트 생성 속도 실시간
SELECT
    toStartOfMinute(event_time) AS minute,
    countIf(event_type = 'NewPart')    AS inserts,
    countIf(event_type = 'MergeParts') AS merges
FROM system.part_log
WHERE event_time >= now() - INTERVAL 10 MINUTE
GROUP BY minute
ORDER BY minute;
```

---

## 해결

### 즉각 조치 — INSERT 일시 중단 후 머지 유도

```sql
-- 1. 백그라운드 머지 강제 실행 (주의: 대용량 테이블은 오래 걸림)
OPTIMIZE TABLE events;  -- 파티션 단위로 나눠서 실행 권장

-- 2. 파트 수 감소 확인
SELECT count() FROM system.parts WHERE table = 'events' AND active;

-- 3. INSERT 재개
```

### 근본 해결 1 — 배치 INSERT

애플리케이션에서 여러 행을 묶어서 한 번에 INSERT.

```python
# 나쁜 예: 건당 INSERT
for event in events:
    client.execute("INSERT INTO events VALUES", [event])

# 좋은 예: 배치 INSERT (최소 1만 행)
BATCH_SIZE = 10000
buffer = []
for event in events:
    buffer.append(event)
    if len(buffer) >= BATCH_SIZE:
        client.execute("INSERT INTO events VALUES", buffer)
        buffer.clear()

if buffer:  # 나머지 처리
    client.execute("INSERT INTO events VALUES", buffer)
```

```sql
-- 배치 크기에 따른 파트 생성 비교:
-- 1건씩 1만 번  → 파트 1만 개 생성
-- 1만건 1번     → 파트 1개 생성 (1만 배 차이)
```

### 근본 해결 2 — Async Insert (배치 불가 시)

여러 애플리케이션 인스턴스에서 소량을 자주 INSERT하는 경우, 서버 사이드 버퍼링.

```sql
-- 유저 단위로 설정 (권장)
ALTER USER app_writer SETTINGS
    async_insert = 1,
    wait_for_async_insert = 1,          -- INSERT가 버퍼에 들어갈 때까지 대기
    async_insert_max_data_size = 1000000,  -- 1MB 차면 플러시
    async_insert_busy_timeout_ms = 1000;   -- 1초마다 강제 플러시
```

```python
# Python 드라이버 예시 (settings 전달)
client.execute(
    "INSERT INTO events VALUES",
    [event],
    settings={
        'async_insert': 1,
        'wait_for_async_insert': 1
    }
)
```

```
Async Insert 흐름:
  Pod1 INSERT(1건) ─┐
  Pod2 INSERT(1건)  ├─→ 서버 메모리 버퍼 누적 → 1MB 또는 1초 후 → 파트 1개 플러시
  Pod3 INSERT(1건) ─┘

  100개 Pod에서 초당 각 100건 = 초당 10,000건
  → 파트 생성: 초당 1~2개 (버퍼 플러시 단위)
```

### wait_for_async_insert 설정 차이

```sql
-- wait_for_async_insert = 0 (기본값)
--   INSERT 응답: 즉시 (버퍼에 들어가자마자)
--   장애 시 데이터 유실 가능

-- wait_for_async_insert = 1 (권장)
--   INSERT 응답: 버퍼→디스크 플러시 완료 후
--   데이터 안전 보장, 약간의 레이턴시 증가
```

---

## 설정 기준 비교

| 상황 | 권장 방법 |
|------|---------|
| Kafka Consumer, ETL | 동기 배치 INSERT (1만 행 단위) |
| 웹 서버 100+ 인스턴스 | Async Insert |
| IoT 디바이스 수만 개 | Async Insert + Buffer Table |
| 배치 작업 (야간 집계) | 동기 INSERT (크기 제한 없음) |

---

## 예방

```sql
-- 모니터링 알람 기준 설정
-- 파트 수 > 300: 경고
-- 파트 수 > 1000: 심각
SELECT table, count() AS parts
FROM system.parts
WHERE active
GROUP BY table
HAVING parts > 300;

-- 적정 INSERT 배치 크기:
-- 최소: 1,000행
-- 권장: 10,000~100,000행
-- 최대: 제한 없지만 한 번에 수백만 행은 메모리 주의
```
