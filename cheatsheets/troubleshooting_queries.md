# 치트시트: 트러블슈팅 쿼리 모음

운영 중 "이상하다" 싶을 때 바로 실행할 수 있는 쿼리들. 증상별로 정리.

---

## 1. INSERT 실패: "Too many parts"

### 진단

```sql
-- 1-A. 어느 테이블이 문제인가?
SELECT database, table,
    count() AS active_parts,
    uniq(partition) AS partitions,
    countIf(level = 0) AS unmerged_parts,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active
GROUP BY database, table
HAVING active_parts > 300
ORDER BY active_parts DESC;

-- 1-B. 파티션별 분포 (과도한 파티셔닝인지)
SELECT partition, count() AS parts, max(level) AS max_level
FROM system.parts
WHERE database = ? AND table = ? AND active
GROUP BY partition
ORDER BY parts DESC
LIMIT 20;

-- 1-C. 머지가 밀리고 있나?
SELECT count() AS running,
    sum(num_parts) AS parts_merging,
    formatReadableSize(sum(total_size_bytes_compressed)) AS size
FROM system.merges;

-- 1-D. 최근 INSERT 패턴 (소량 빈번인지)
SELECT
    toStartOfMinute(event_time) AS minute,
    count() AS inserts,
    avg(written_rows) AS avg_rows_per_insert
FROM system.query_log
WHERE query_kind = 'Insert'
  AND event_time >= now() - INTERVAL 30 MINUTE
GROUP BY minute
ORDER BY minute DESC;
```

### 임시 응급 처치

```sql
-- 임계값 임시 상향 (파일 설정이 아니면 재시작 시 사라짐)
-- 장기 해결책은 INSERT 전략 수정 또는 파티션 재설계

-- 강제 머지 트리거 (조심스럽게)
OPTIMIZE TABLE mydb.mytable;
-- FINAL 붙이지 말 것. 한 사이클만 실행됨
```

---

## 2. 쿼리가 느림

### 진단

```sql
-- 2-A. 가장 느린 최근 쿼리 Top 10
SELECT
    query_id,
    query_duration_ms AS ms,
    formatReadableSize(memory_usage) AS mem,
    read_rows, read_bytes,
    substring(query, 1, 100) AS q
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date >= today()
  AND query_kind = 'Select'
ORDER BY query_duration_ms DESC
LIMIT 10;

-- 2-B. 특정 쿼리의 인덱스 효과 확인
EXPLAIN indexes = 1
SELECT ... FROM mytable WHERE ...;
-- 봐야 할 곳:
--   MinMax:     Parts: A/B
--   Partition:  Parts: A/B
--   PrimaryKey: Parts: A/B, Granules: C/D
-- C/D 비율이 1에 가까우면 인덱스 미작동

-- 2-C. 테이블 스캔 vs 인덱스 스캔 비교
EXPLAIN PIPELINE
SELECT ... FROM mytable WHERE ...;

-- 2-D. 실제 실행 프로파일 (query_log ProfileEvents)
SELECT
    query_duration_ms,
    ProfileEvents['SelectedRows'] AS selected_rows,
    ProfileEvents['SelectedBytes'] AS selected_bytes,
    ProfileEvents['OSReadChars'] AS os_read,
    ProfileEvents['RealTimeMicroseconds'] AS cpu_us,
    ProfileEvents['UserTimeMicroseconds'] AS user_us
FROM system.query_log
WHERE query_id = '<query_id>'
  AND type = 'QueryFinish';
```

### 자주 있는 원인

| 증상 | 쿼리 결과 | 원인 | 해결 |
|------|----------|------|------|
| `Granules: Y/Y` (동일) | 풀 스캔 | ORDER BY 키와 WHERE 불일치 | 키 재설계 또는 projection |
| `Parts: 수백` | 머지 지연 | level 0 파트 많음 | 배치 크기 증가 |
| `peak_memory_usage` 높음 | 메모리 과소비 | GROUP BY 고카디널리티 | 샘플링, pre-aggregation |
| `read_bytes` 높음 | 많이 읽음 | `SELECT *` 사용 | 필요 컬럼만 |
| 타임아웃 | 서버 부하 | 동시 쿼리 많음 | `max_concurrent_queries` 조정 |

---

## 3. 레플리카 지연

### 진단

```sql
-- 3-A. 지연 중인 레플리카
SELECT database, table, replica_name,
    is_leader, absolute_delay, queue_size,
    inserts_in_queue, merges_in_queue,
    last_queue_update
FROM system.replicas
WHERE absolute_delay > 10
ORDER BY absolute_delay DESC;

-- 3-B. 복제 큐에 쌓인 작업
SELECT type, count()
FROM system.replication_queue
GROUP BY type;

-- 3-C. Keeper 상태
SELECT * FROM system.zookeeper
WHERE path = '/clickhouse/tables/shard01/mytable/replicas';
```

### 해결 가이드

```
inserts_in_queue 높음
  → 네트워크 대역폭 확인
  → 파트 다운로드 병목

merges_in_queue 높음
  → background_pool_size 증가
  → 디스크 I/O 확인

queue_size 계속 증가
  → Keeper 연결 이상
  → 레플리카 재시작 고려
```

---

## 4. Mutation이 안 끝남

```sql
-- 4-A. 진행 중인 mutation
SELECT database, table, mutation_id, command,
    create_time, is_done, parts_to_do, latest_fail_reason,
    latest_fail_time
FROM system.mutations
WHERE is_done = 0
ORDER BY create_time;

-- 4-B. 문제 mutation 취소
KILL MUTATION WHERE mutation_id = '...';

-- 4-C. 실패 원인 확인
SELECT latest_fail_reason
FROM system.mutations
WHERE mutation_id = '...';
```

### 안 끝나는 원인

- `parts_to_do` 감소 X → 다른 mutation이 큐 앞에서 막혀 있음
- `latest_fail_reason`에 에러 있음 → 문법/타입 에러일 가능성
- mutation 삭제해도 재시작 시 복구됨 → KILL MUTATION 필수

---

## 5. 디스크 공간 부족

```sql
-- 5-A. 테이블별 용량 Top 10
SELECT database, table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    count() AS parts,
    countIf(NOT active) AS inactive_parts,
    formatReadableSize(sumIf(bytes_on_disk, NOT active)) AS inactive_size
FROM system.parts
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
LIMIT 10;

-- 5-B. 압축률 확인 (LowCardinality, 코덱 점검용)
SELECT database, table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE active
GROUP BY database, table
HAVING compressed > 1e9
ORDER BY compressed DESC;

-- 5-C. inactive 파트가 많으면 확인
SELECT name, bytes_on_disk, modification_time, remove_time
FROM system.parts
WHERE NOT active
  AND database = ? AND table = ?
ORDER BY bytes_on_disk DESC;

-- 5-D. 임시 해결: old_parts_lifetime 단축
ALTER TABLE ? MODIFY SETTING old_parts_lifetime = 60;  -- 8분 → 1분
```

---

## 6. 메모리 부족 (OOM, MEMORY_LIMIT_EXCEEDED)

```sql
-- 6-A. 메모리 많이 쓴 최근 쿼리
SELECT query_id,
    formatReadableSize(peak_memory_usage) AS peak,
    formatReadableSize(memory_usage) AS current,
    query
FROM system.query_log
WHERE event_date >= today()
  AND peak_memory_usage > 1e9
ORDER BY peak_memory_usage DESC
LIMIT 20;

-- 6-B. 현재 실행 중 쿼리 메모리
SELECT query_id,
    formatReadableSize(memory_usage) AS mem,
    elapsed,
    substring(query, 1, 100) AS q
FROM system.processes
ORDER BY memory_usage DESC;

-- 6-C. 서버 전체 메모리 사용
SELECT formatReadableSize(value)
FROM system.asynchronous_metrics
WHERE metric IN ('OSMemoryAvailable', 'MemoryResident', 'MemoryVirtual');

-- 6-D. 문제 쿼리 강제 종료
KILL QUERY WHERE query_id = '...';
```

### 흔한 원인

- 고카디널리티 `GROUP BY` (유저 수억 × 제품 수천)
- `DISTINCT` (uniq로 대체)
- 대용량 `JOIN` (빌드 측이 메모리에 올라감)
- `ORDER BY` 대용량 (external sort 설정 필요)

---

## 7. 디스크 I/O 병목

```sql
-- 7-A. OS 레벨 I/O 통계
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric LIKE 'OSIOWait%' OR metric LIKE '%Disk%'
ORDER BY metric;

-- 7-B. 최근 읽기가 많은 쿼리
SELECT
    formatReadableSize(ProfileEvents['OSReadBytes']) AS os_read,
    formatReadableSize(ProfileEvents['ReadBufferFromFileDescriptorReadBytes']) AS file_read,
    query_duration_ms,
    query
FROM system.query_log
WHERE event_time >= now() - INTERVAL 1 HOUR
  AND ProfileEvents['OSReadBytes'] > 1e9
ORDER BY ProfileEvents['OSReadBytes'] DESC
LIMIT 10;
```

---

## 8. 네트워크/Keeper 문제

```sql
-- 8-A. Keeper 연결 상태
SELECT name, host, port, status, last_exception
FROM system.clusters;

-- 8-B. Keeper 세션 확인
SELECT * FROM system.zookeeper_connection;

-- 8-C. DNS/분산 쿼리 에러
SELECT event_time, query, exception
FROM system.query_log
WHERE type = 'ExceptionWhileProcessing'
  AND event_date >= today()
ORDER BY event_time DESC
LIMIT 20;
```

---

## 9. 만능 응급 스냅샷

문제가 생기면 먼저 이걸 돌려서 전체 상태를 캡처:

```sql
SELECT 'parts' AS metric, count() AS value FROM system.parts WHERE active
UNION ALL
SELECT 'merges', count() FROM system.merges
UNION ALL
SELECT 'mutations', count() FROM system.mutations WHERE NOT is_done
UNION ALL
SELECT 'queries', count() FROM system.processes
UNION ALL
SELECT 'replicas_delayed', count() FROM system.replicas WHERE absolute_delay > 10
UNION ALL
SELECT 'keeper_session_uptime', max(session_uptime_elapsed_seconds) FROM system.zookeeper_connection;
```

---

## 10. 예방적 모니터링 쿼리

Prometheus/Grafana에서 주기적으로 수집:

```sql
-- 10-A. 파트 수 추이 (1분마다)
SELECT count() FROM system.parts WHERE active;

-- 10-B. 머지 큐 (1분마다)
SELECT count() FROM system.merges;

-- 10-C. 지연 레플리카 수
SELECT count() FROM system.replicas WHERE absolute_delay > 10;

-- 10-D. 큐 쌓인 INSERT
SELECT count() FROM system.processes WHERE query ILIKE 'INSERT%';

-- 10-E. 에러 쿼리 비율 (5분 윈도우)
SELECT
    countIf(type != 'QueryFinish') AS errors,
    count() AS total,
    errors / total AS error_rate
FROM system.query_log
WHERE event_time >= now() - INTERVAL 5 MINUTE;
```
