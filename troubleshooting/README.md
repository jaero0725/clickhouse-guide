# 트러블슈팅 케이스 스터디

실무에서 자주 발생하는 ClickHouse 실수와 해결 과정을 케이스별로 정리.
각 케이스는 **증상 → 원인 → 진단 → 해결 → 예방** 순서로 구성된다.

---

## 케이스 목록

### A. 파티셔닝 실수

| 케이스 | 증상 | 핵심 키워드 |
|--------|------|------------|
| [A1. 일 단위 파티셔닝](A1_daily_partition_explosion.md) | 파트 수 폭증, 머지 불가, 쓰기 중단 | Too many parts, PARTITION BY toDate |
| [A2. 고카디널리티 파티셔닝](A2_high_cardinality_partition.md) | 파티션 수천 개, DROP 불가, 메모리 부족 | user_id 파티셔닝, 카디널리티 |

### B. 엔진 오용

| 케이스 | 증상 | 핵심 키워드 |
|--------|------|------------|
| [B1. ReplacingMergeTree + FINAL 없이 조회](B1_replacing_without_final.md) | 중복 행 반환, 집계 값 2배 | ReplacingMergeTree, FINAL, argMax |
| [B2. CollapsingMergeTree sign 순서 꼬임](B2_collapsing_sign_order.md) | 잔액/수량이 음수 또는 0, 데이터 소실 | CollapsingMergeTree, sign=-1 |
| [B3. SummingMergeTree String 컬럼 소실](B3_summing_string_columns.md) | 머지 후 String 컬럼이 임의 값으로 | SummingMergeTree, 비집계 컬럼 |

### C. 쿼리 실수

| 케이스 | 증상 | 핵심 키워드 |
|--------|------|------------|
| [C1. SELECT * 남용](C1_select_star.md) | 예상보다 10~100배 느린 쿼리 | 컬럼 지향, 불필요한 I/O |
| [C2. FINAL 남발](C2_final_overuse.md) | 쿼리가 싱글 스레드, CPU 폭증 | FINAL, 병렬 처리 불가 |
| [C3. Distributed GROUP BY OOM](C3_distributed_group_by_oom.md) | 쿼리 중 OOM, Memory limit exceeded | Distributed, 고카디널리티 GROUP BY |
| [C4. DISTINCT vs uniq](C4_distinct_vs_uniq.md) | DAU 쿼리 10초 이상, 메모리 수 GB | DISTINCT, uniq, HyperLogLog |

### D. 애플리케이션 단 실수

| 케이스 | 증상 | 핵심 키워드 |
|--------|------|------------|
| [D1. 건당 INSERT](D1_per_row_insert.md) | Too many parts, DB 응답 불가 | 파트 폭증, Async Insert |
| [D2. Distributed 테이블에 INSERT](D2_distributed_insert.md) | 쓰기 지연, 이중 네트워크 홉 | Distributed INSERT, 로컬 테이블 |
| [D3. Mutation 남발](D3_mutation_overuse.md) | 백그라운드 머지 중단, 디스크 2배 사용 | ALTER TABLE UPDATE/DELETE, Mutation |

---

## 빠른 진단 — 증상으로 찾기

```
쓰기가 갑자기 느려지거나 에러 발생
  ├── "Too many parts" 에러           → D1, A1
  ├── INSERT 타임아웃                 → D1, D2
  └── 디스크 사용량 급증              → D3

읽기/쿼리 문제
  ├── 쿼리가 예상보다 10배 이상 느림  → C1, C2, A1
  ├── OOM / Memory limit exceeded    → C3, C4
  └── 결과값이 이상함 (중복, 음수)    → B1, B2, B3

데이터 정합성 문제
  ├── 중복 행 조회됨                  → B1
  ├── 집계 값이 2배 또는 0            → B2, B3
  └── 머지 후 컬럼 값 변경됨          → B3
```

---

## 공통 진단 쿼리

```sql
-- 1. 테이블별 파트 수 현황
SELECT table, count() AS parts, sum(rows) AS total_rows
FROM system.parts
WHERE database = currentDatabase() AND active
GROUP BY table ORDER BY parts DESC;

-- 2. 파티션별 파트 수 (10 넘으면 머지 지연 의심)
SELECT partition, count() AS parts
FROM system.parts
WHERE table = 'your_table' AND active
GROUP BY partition ORDER BY parts DESC LIMIT 20;

-- 3. 현재 실행 중인 Mutation 목록
SELECT table, command, parts_to_do, is_done, create_time
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time;

-- 4. 최근 느린 쿼리 (1초 이상)
SELECT query_start_time, query_duration_ms, query
FROM system.query_log
WHERE type = 'QueryFinish' AND query_duration_ms > 1000
ORDER BY query_duration_ms DESC LIMIT 10;

-- 5. 백그라운드 머지 현황
SELECT table, elapsed, progress, num_parts
FROM system.merges
ORDER BY elapsed DESC;
```
