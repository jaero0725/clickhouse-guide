# 11장. 모니터링과 트러블슈팅

앞선 10개 장에서 배운 아키텍처 지식을 **운영 현장에서 활용하는 방법**을 정리한다. ClickHouse의 system 테이블은 서버 내부 상태를 관찰하는 강력한 창이며, 대부분의 문제는 이 테이블들의 쿼리로 진단할 수 있다.

---

## 11.1 핵심 System 테이블 맵

| System 테이블 | 관찰 대상 | 관련 장 |
|--------------|-----------|---------|
| `system.parts` | 파트 현황: 활성 파트 수, 크기, 레벨, 파티션 분포 | 4, 5장 |
| `system.merges` | 진행 중인 머지: 진행률, 소요 시간, 파트 수 | 6장 |
| `system.mutations` | 진행 중인 mutation: 상태, 에러, 대기열 | 10장 |
| `system.replicas` | 레플리카 상태: lag, 큐 크기, 리더 여부 | 8장 |
| `system.query_log` | 쿼리 이력: 성능, 메모리, 읽기량, 에러 | 2, 3장 |
| `system.part_log` | 파트 생명주기: INSERT, 머지, 삭제 이벤트 | 4, 6장 |
| `system.metrics` | 실시간 메트릭: 스레드, 커넥션, 메모리 | 3장 |
| `system.asynchronous_metrics` | 주기적 메트릭: 디스크, OS 수준 통계 | - |
| `system.events` | 누적 카운터: 읽기/쓰기 바이트, 쿼리 수 | - |
| `system.zookeeper` | Keeper 상태: 노드, 자식 수, 경로 탐색 | 8장 |

---

## 11.2 문제별 진단 가이드

### 문제 1: "Too many parts" 에러

INSERT가 거부되는 가장 흔한 운영 사고.

**원인 진단:**

```sql
-- 1단계: 어느 테이블이 문제인지 확인
SELECT
    database,
    `table`,
    count() AS active_parts,
    uniq(partition) AS partitions,
    countIf(level = 0) AS unmerged_parts
FROM system.parts
WHERE active
GROUP BY database, `table`
HAVING active_parts > 300
ORDER BY active_parts DESC;
```

```sql
-- 2단계: 파티션별 분포 확인 (과도한 파티셔닝 여부)
SELECT
    partition,
    count() AS parts,
    min(level) AS min_level,
    max(rows) AS max_rows
FROM system.parts
WHERE database = 'mydb' AND `table` = 'mytable' AND active
GROUP BY partition
ORDER BY parts DESC
LIMIT 20;
```

```sql
-- 3단계: 머지가 밀리고 있는지 확인
SELECT count() AS running_merges,
       sum(num_parts) AS merging_parts,
       formatReadableSize(sum(total_size_bytes_compressed)) AS total_size
FROM system.merges;
```

**원인별 대응:**

| 원인 | 진단 지표 | 대응 |
|------|-----------|------|
| 소량 빈번 INSERT | `level = 0` 파트가 대부분 | 배칭 크기 증가, async insert |
| 과도한 파티셔닝 | `uniq(partition)` >> 1000 | 파티션 키 재설계 (새 테이블 필요) |
| 머지 속도 부족 | `system.merges` 항상 바쁨 | `background_pool_size` 증가 |
| Mutation 점유 | `system.mutations`에 `is_done=0` 다수 | KILL MUTATION 후 대안 사용 |
| 디스크 I/O 병목 | OS 레벨 I/O wait 높음 | 스토리지 업그레이드 (HDD→SSD) |

---

### 문제 2: 쿼리 성능 저하

```sql
-- 느린 쿼리 Top 10
SELECT
    query_id,
    query_duration_ms,
    read_rows,
    formatReadableSize(read_bytes) AS read_size,
    formatReadableSize(peak_memory_usage) AS peak_mem,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND event_date >= today() - 1
ORDER BY query_duration_ms DESC
LIMIT 10;
```

```sql
-- 특정 쿼리의 인덱스 skip 효과 확인
EXPLAIN indexes = 1
SELECT ... FROM mytable WHERE ...;
-- Granules: X/Y → X/Y 비율이 높으면 인덱스 비효율
```

**원인별 대응:**

| 증상 | 진단 | 대응 |
|------|------|------|
| 전체 스캔 (Granules: Y/Y) | EXPLAIN에서 PrimaryKey 없음 | ORDER BY 키 재설계 (7장) |
| 파트가 너무 많음 | Parts: 수백 | 파티셔닝 재검토, 머지 대기 |
| 메모리 부족 | peak_memory_usage 과다 | `max_memory_usage` 조정, 쿼리 최적화 |
| 대량 데이터 읽기 | read_bytes 과다 | 필요 컬럼만 SELECT, PREWHERE 활용 |

---

### 문제 3: 레플리카 지연 (Replication Lag)

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

**`absolute_delay`**: 가장 최근 레플리카 대비 지연 시간(초). 증가 추세면 레플리카가 따라잡지 못하는 것.

**대응:**
- `inserts_in_queue` 높음 → 네트워크 대역폭 확인. 파트 다운로드 병목
- `merges_in_queue` 높음 → 머지 스레드 부족. `background_pool_size` 증가
- Keeper 연결 문제 → `system.zookeeper` 상태 확인

---

### 문제 4: Mutation이 끝나지 않음

```sql
SELECT
    database,
    `table`,
    mutation_id,
    command,
    create_time,
    is_done,
    parts_to_do,
    latest_fail_reason
FROM system.mutations
WHERE is_done = 0
ORDER BY create_time ASC;
```

`parts_to_do`가 줄지 않으면 mutation이 진행되지 않는 것. `latest_fail_reason`에서 원인을 확인하라.

**대응:**
```sql
-- 문제 mutation 강제 취소
KILL MUTATION WHERE mutation_id = 'mutation_0000000001';
```

---

### 문제 5: 디스크 공간 부족

```sql
-- 테이블별 디스크 사용량
SELECT
    database,
    `table`,
    formatReadableSize(sum(bytes_on_disk)) AS total_size,
    count() AS parts,
    countIf(active = 0) AS inactive_parts,
    formatReadableSize(sumIf(bytes_on_disk, active = 0)) AS inactive_size
FROM system.parts
GROUP BY database, `table`
ORDER BY sum(bytes_on_disk) DESC
LIMIT 20;
```

**inactive 파트**가 과도하면 `old_parts_lifetime` (기본 8분) 이 경과하지 않았거나, 머지가 진행 중인 것. 머지 중에는 원본 + 결과 파트가 동시 존재하므로 일시적으로 2배 공간이 필요하다.

---

## 11.3 상시 모니터링 대시보드 쿼리

프로덕션 환경에서 주기적으로 확인해야 할 핵심 쿼리:

### 서버 전반 상태

```sql
-- 실시간 스레드 현황
SELECT metric, value
FROM system.metrics
WHERE metric IN (
    'Query', 'Merge', 'PartMutation',
    'GlobalThread', 'GlobalThreadActive',
    'BackgroundMergesAndMutationsPoolTask',
    'ReplicatedSend', 'ReplicatedFetch'
)
ORDER BY metric;
```

### 파트 건강 상태 (모든 테이블)

```sql
SELECT
    database,
    `table`,
    count() AS active_parts,
    countIf(level = 0) AS level0_parts,
    round(countIf(level = 0) / count() * 100, 1) AS level0_pct,
    max(modification_time) AS last_insert
FROM system.parts
WHERE active
GROUP BY database, `table`
HAVING active_parts > 10
ORDER BY level0_pct DESC;
-- level0_pct가 높으면 머지가 밀리고 있다는 신호
```

### 쿼리 성능 추이

```sql
-- 시간대별 평균 쿼리 성능
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS queries,
    round(avg(query_duration_ms)) AS avg_ms,
    round(quantile(0.95)(query_duration_ms)) AS p95_ms,
    formatReadableSize(avg(read_bytes)) AS avg_read
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND event_date >= today() - 1
GROUP BY hour
ORDER BY hour;
```

---

## 11.4 설정 변경 확인

2장에서 배운 Configuration 리로드 특성을 떠올리자: 설정 파일에 오류가 있으면 서버는 **이전 설정을 유지**하고 경고만 남긴다. 변경이 적용되었는지 반드시 확인하라.

```sql
-- 현재 적용된 설정 확인
SELECT name, value, changed
FROM system.settings
WHERE name IN ('max_threads', 'background_pool_size',
               'async_insert', 'insert_quorum')
ORDER BY name;

-- 현재 머지 관련 설정
SELECT name, value
FROM system.merge_tree_settings
WHERE name IN ('max_bytes_to_merge_at_max_space_in_pool',
               'parts_to_throw_insert',
               'old_parts_lifetime',
               'index_granularity')
ORDER BY name;
```

---

## 11.5 ClickHouse 24.10+ 내장 대시보드

ClickHouse 24.10부터 `/dashboard` HTTP 핸들러에서 내장 모니터링 대시보드를 제공한다. 브라우저에서 `http://clickhouse-server:8123/dashboard`로 접근 가능.

추가로 `/merges` 핸들러에서 머지 현황을 시각적으로 확인할 수 있다 (6장 참조).

---

## 11.6 Key Takeaways

| 영역 | 핵심 모니터링 포인트 |
|------|---------------------|
| **파트 건강** | `system.parts`에서 active_parts 수, level=0 비율, 파티션 분포 |
| **머지 상태** | `system.merges`에서 running count, progress, 소요 시간 |
| **레플리카** | `system.replicas`에서 absolute_delay, queue_size |
| **쿼리 성능** | `system.query_log`에서 duration, read_bytes, peak_memory |
| **Mutation** | `system.mutations`에서 is_done=0, parts_to_do, fail_reason |
| **스레드** | `system.metrics`에서 GlobalThread, Active thread 수 |
| **디스크** | `system.parts`에서 bytes_on_disk, inactive 파트 크기 |
| **설정** | `system.settings`에서 변경 적용 여부 확인 |

---

## 11.7 트러블슈팅 플로우차트

```
문제 발생
  │
  ├── INSERT 실패?
  │     ├── "Too many parts" → §11.2 문제 1
  │     └── 타임아웃 → 배치 크기 확인, 네트워크 확인
  │
  ├── 쿼리가 느리다?
  │     ├── EXPLAIN indexes=1 → Granule skip 확인 → §11.2 문제 2
  │     ├── system.query_log → read_bytes, peak_memory 확인
  │     └── 파트 수 확인 → 머지 밀림 여부
  │
  ├── 레플리카 지연?
  │     └── system.replicas → §11.2 문제 3
  │
  ├── 디스크 부족?
  │     └── inactive 파트 + 머지 중 임시 공간 → §11.2 문제 5
  │
  └── Mutation 멈춤?
        └── system.mutations → KILL MUTATION → §11.2 문제 4
```

---

*이것으로 ClickHouse 학습 가이드 전 11장을 마칩니다.*

## 부록: 참고 자료

- **공식 Architecture 문서**: `/development/architecture` (이 가이드의 주 참조 소스)
- **VLDB 2024 논문**: "ClickHouse - Lightning Fast Analytics for Everyone" (학술적 아키텍처 개요)
- **공식 Best Practices 시리즈**: choosing-a-primary-key, choosing-a-partitioning-key, selecting-an-insert-strategy, avoid-mutations, avoid-optimize-final
- **Core Concepts 시리즈**: /parts, /partitions, /merges, /primary-indexes, /shards
- **ClickHouse SQL Playground**: sql.clickhouse.com (예제 쿼리 실행 가능)
