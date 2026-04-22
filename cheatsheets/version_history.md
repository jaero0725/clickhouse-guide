# ClickHouse 버전 히스토리

> ClickHouse는 매달 새 버전을 출시한다. 버전 번호 형식: `YY.MAJOR.MINOR.PATCH`
> 예) `24.8.3.1` = 2024년, 8번째 메이저 릴리스, 패치 3.1

---

## 버전 정책

```
┌─────────────────────────────────────────────────────────────┐
│  일반 릴리스 (매월)           LTS 릴리스 (년 2회: 3월, 8월)  │
│  지원 기간: 3개월              지원 기간: 12개월             │
│  ex) 24.1, 24.2, ...         ex) 24.3 LTS, 24.8 LTS       │
└─────────────────────────────────────────────────────────────┘
```

**프로덕션 권장**: LTS 버전 사용
- 현재 활성 LTS: **24.8**, **25.3**
- 최신 안정: **25.12** (2025년 12월 기준)

---

## 빠른 참조 — 버전별 핵심 변경

| 버전 | 출시 | LTS | 핵심 변경 |
|------|------|-----|---------|
| 23.3 | 2023.03 | ✅ | Parallel Replicas (샤드 내 병렬 처리) |
| 23.8 | 2023.08 | ✅ | S3Queue 엔진, 인버티드 인덱스(실험) |
| 24.1 | 2024.01 | - | Punycode 함수, HTTP 출력 최적화 |
| 24.3 | 2024.03 | ✅ | **Query Analyzer GA**, JOIN 성능 5x |
| 24.6 | 2024.06 | - | 최적 테이블 정렬 자동화 |
| 24.8 | 2024.08 | ✅ | **JSON 타입**, TimeSeries 엔진, Kafka 정확히-1회 |
| 24.12 | 2024.12 | - | 재귀 CTE, QUALIFY 절 |
| 25.1 | 2025.01 | - | Merge 테이블 Variant 타입 통합 |
| 25.3 | 2025.03 | ✅ | **쿼리 조건 캐시** (반복 필터 즉시 응답) |
| 25.4 | 2025.04 | - | **Lazy Materialization** (I/O 40x, 메모리 300x 절감) |
| 25.5 | 2025.05 | - | 상관 서브쿼리 WHERE/SELECT 지원, TPC-H 커버 |
| 25.6 | 2025.06 | - | Time/Time64 타입, SELECT 단일 스냅샷 |
| 25.7 | 2025.07 | - | **Lightweight UPDATE** (패치 파트, 컬럼 전체 재기록 X) |
| 25.8 | 2025.08 | ✅ | **Vector Search GA**, Iceberg 쓰기 지원 |
| 25.9 | 2025.09 | - | Text Index (실험), 전문 검색 개선 |
| 25.10 | 2025.10 | - | 인버티드 인덱스 재설계(RAM 초과 데이터셋), Apache Paimon |
| 25.11 | 2025.11 | - | GROUP BY 없이 HAVING 사용 (ANSI SQL 호환) |
| 25.12 | 2025.12 | - | 최적 JOIN 순서 탐색(DPSize), HMAC 함수 |

---

## 상세 — 2023년 (23.x)

### 23.1 — 2023년 1월
- **Query Result Cache**: 동일 쿼리 재실행 시 결과 캐시 반환
- 인버티드 인덱스 실험적 지원 (데이터 스킵 인덱스)
- 파라미터 전달 기반 동적 뷰

### 23.3 LTS — 2023년 3월 ⭐ LTS
- **Parallel Replicas**: 동일 샤드의 여러 레플리카로 데이터 병렬 처리
  ```sql
  SET max_parallel_replicas = 3;
  -- 같은 샤드의 3개 레플리카가 데이터를 나눠서 처리
  ```
- 메모리 내 마크 압축 → 인덱스 메모리 절감
- 취소된 쿼리의 부분 결과 확인 가능

### 23.8 LTS — 2023년 8월 ⭐ LTS
- **S3Queue 엔진**: S3에 올라오는 파일 자동 구독 + MV로 변환
  ```sql
  CREATE TABLE s3_queue (...)
  ENGINE = S3Queue('s3://bucket/path/*.csv', 'CSV')
  SETTINGS mode = 'ordered';  -- 순서 보장
  ```
- Inverted Index 실험적 개선 (전문 검색)

---

## 상세 — 2024년 (24.x)

### 24.1 — 2024년 1월
- Punycode 인코딩/디코딩 함수
- Datadog 분위수 스케치 함수
- HTTP 출력 속도 개선, Parallel Replicas 성능 향상

### 24.3 LTS — 2024년 3월 ⭐ LTS
> **가장 중요한 24.x 업그레이드 포인트**

- **Query Analyzer (쿼리 분석기) 기본 활성화**
  - 이전 쿼리 파이프라인 대비 정확도, 최적화 수준 대폭 향상
  - 복잡한 JOIN, 서브쿼리 처리 개선
  - 마이그레이션 시 기존 쿼리 동작 변화 주의 (아래 주의사항 참고)
  ```sql
  -- 이전으로 돌리려면 (임시):
  SET enable_analyzer = 0;
  ```
- **JOIN 성능 5배 향상**: 대규모 JOIN 쿼리
- JOIN에서 SAMPLE, FINAL 지원
- S3 Express One Zone Storage 지원
- 프라이머리 키 불필요 컬럼 로딩 방지 옵션

### 24.5 — 2024년 5월
- CROSS JOIN 메모리 내 압축
- CROSS JOIN 디스크 처리 (Right-side가 메모리 초과 시)
- S3에서 tar/zip/7z 컨테이너 형식 직접 읽기

### 24.6 — 2024년 6월
- **최적 테이블 정렬 자동화**: 데이터 수집 시 ORDER BY 키 기준 자동 최적 정렬
  → 압축률 향상

### 24.8 LTS — 2024년 8월 ⭐ LTS
> **현재 가장 널리 쓰이는 LTS**

- **새로운 JSON 타입** (완전히 재설계)
  ```sql
  CREATE TABLE events (
    id UInt64,
    data JSON                      -- 동적 JSON 필드 지원
  ) ENGINE = MergeTree ORDER BY id;

  -- 접근법
  SELECT data.user_id, data.action FROM events;
  ```
- **TimeSeries 엔진**: Prometheus 메트릭 저장 최적화
  ```sql
  CREATE TABLE metrics ENGINE = TimeSeries;
  ```
- **Kafka Exactly-Once Processing**: 중복 없는 Kafka 메시지 처리
  - Keeper에서 오프셋 관리 옵션
- Projection 동기화 설정 (deduplication 머지 시 동작 제어)

### 24.9 ~ 24.11 — 2024년 9~11월
- 재귀 CTE (트리/그래프 순회)
  ```sql
  WITH RECURSIVE tree AS (
    SELECT id, parent_id, name FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, c.name FROM categories c
    JOIN tree t ON c.parent_id = t.id
  )
  SELECT * FROM tree;
  ```
- QUALIFY 절: 윈도우 함수 결과 필터링
  ```sql
  SELECT user_id, event, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts)
  FROM events
  QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts) = 1;
  ```

### 24.12 — 2024년 12월
- `TRUNCATE ALL TABLES` 명령 추가
- `DROP DETACHED PARTITION ALL` — 모든 분리 파티션 일괄 삭제

---

## 상세 — 2025년 (25.x)

### 25.1 — 2025년 1월
- Merge 테이블: 모든 테이블의 컬럼을 Variant 타입으로 통합
  ```sql
  -- 이전: 첫 번째 테이블 구조만 사용
  -- 이후: 컬럼 타입을 공통 타입 또는 Variant로 자동 통합
  CREATE TABLE merged ENGINE = Merge(currentDatabase(), 'table_.*');
  ```

### 25.3 LTS — 2025년 3월 ⭐ LTS
> **현재 최신 LTS**

- **쿼리 조건 캐시 (Query Condition Cache)**
  - WHERE 절 조건이 어떤 그래뉼을 만족하는지 기억
  - 반복적인 대시보드/알림 쿼리에서 즉시 응답
  ```sql
  -- 캐시 상태 확인
  SELECT * FROM system.query_condition_cache;
  ```

### 25.4 — 2025년 4월
- **Lazy Materialization** (핵심 성능 향상)
  - 컬럼 데이터를 실제로 필요한 시점까지 읽기 지연
  - 테스트: 1,576배 속도 향상, I/O 40배 감소, 메모리 300배 감소
  ```
  기존: WHERE 조건 → 모든 컬럼 읽기 → 필터 → 결과
  이후: WHERE 조건 → "읽어야 할 행 목록" 추적 → 필요한 컬럼만 읽기
  ```
- 히스토그램 기반 레이턴시 추적 (Observability Dashboard)

### 25.5 — 2025년 5월
- 상관 서브쿼리 (WHERE/SELECT 내부) 지원
  ```sql
  -- 이제 가능:
  SELECT user_id, (SELECT max(amount) FROM orders o WHERE o.user_id = u.user_id)
  FROM users u
  WHERE (SELECT count() FROM orders o WHERE o.user_id = u.user_id) > 5;
  ```
- TPC-H 벤치마크 쿼리 수정 없이 실행 가능
- 119개 신규 함수 중 18개가 이 버전에서 추가

### 25.6 — 2025년 6월
- **Time / Time64 타입**: 시간값 (절대 날짜 없이)
  ```sql
  CREATE TABLE schedule (
    event_name String,
    start_time Time,       -- HH:MM:SS
    duration   Time64(3)   -- 밀리초 정밀도
  ) ENGINE = MergeTree ORDER BY start_time;
  ```
- SELECT 단일 스냅샷 보장
- 여러 Projection 필터링 지원

### 25.7 — 2025년 7월
- **Lightweight UPDATE (경량 업데이트)** — 가장 큰 변화 중 하나
  ```sql
  -- 이전: ALTER TABLE ... UPDATE → 전체 컬럼 재기록 (Mutation)
  -- 이후: 표준 UPDATE → 패치 파트만 기록
  UPDATE orders SET status = 'shipped' WHERE order_id = 12345;
  ```
  - 패치 파트(patch part)만 기록 → 즉시 반영
  - 기존 Mutation 대비 쓰기 영향 최소화
  - 지원 엔진: MergeTree, ReplacingMergeTree, CollapsingMergeTree (+ Replicated)

### 25.8 LTS — 2025년 8월 ⭐ LTS (예정)
- **벡터 유사도 검색 GA (Vector Similarity Search)**
  ```sql
  CREATE TABLE embeddings (
    id     UInt64,
    vector Array(Float32),
    INDEX  vec_idx vector TYPE vector_similarity('hnsw', 'cosineDistance')
  ) ENGINE = MergeTree ORDER BY id;

  -- k-NN 검색
  SELECT id FROM embeddings
  ORDER BY cosineDistance(vector, [0.1, 0.2, ...])
  LIMIT 10;
  ```
  - Binary Quantization: 메모리 절감 + 인덱스 빌드 가속
- **Apache Iceberg 쓰기 지원**: 기존 Iceberg 테이블에 INSERT
- Delta Lake INSERT 지원

### 25.9 — 2025년 9월
- Text Index 실험적 도입 (전문 검색 특화)

### 25.10 — 2025년 10월
- **인버티드 인덱스 완전 재설계**
  - RAM을 초과하는 데이터셋도 처리 가능
  ```sql
  ALTER TABLE logs ADD INDEX content_idx content TYPE full_text;
  ```
- Apache Paimon 테이블 포맷 지원
- Iceberg: ALTER UPDATE, 분산 쓰기 완성 (읽기/쓰기 동등)
- ZooKeeper 운영 통계, CPU/메모리 자동 알림

### 25.11 — 2025년 11월
- GROUP BY 없이 HAVING 사용 가능 (ANSI SQL 호환)
  ```sql
  -- 이제 가능 (PostgreSQL 호환):
  SELECT sum(amount) FROM orders HAVING sum(amount) > 1000000;
  ```

### 25.12 — 2025년 12월
- **DPSize JOIN 순서 최적화**: 최적 JOIN 순서 탐색 알고리즘
- HMAC 함수 (메시지 무결성/인증 검증)
- Text Index → Beta 승격

---

## 마이그레이션 주의사항

### 23.x → 24.3 (Query Analyzer 활성화)
```sql
-- 영향 받는 케이스:
-- 1. 모호한 컬럼명을 가진 쿼리
SELECT a FROM t1 JOIN t2 USING (id);  -- 이제 명시적으로 t1.a 또는 t2.a 필요할 수 있음

-- 2. 이전에 "우연히 동작하던" 쿼리가 오류로 바뀔 수 있음

-- 확인 방법:
SET enable_analyzer = 0;  -- 구 동작으로 임시 롤백
-- 문제 쿼리 발견 후 수정

-- 공식 마이그레이션 가이드:
-- https://clickhouse.com/docs/guides/developer/query-analyzer-migration
```

### 24.x → 25.7 (Lightweight UPDATE)
```sql
-- 기존 ALTER ... UPDATE 쿼리는 계속 동작 (하위 호환)
-- 새 UPDATE 문법 사용 시 패치 파트 방식으로 자동 처리
-- 주의: UPDATE 후 SELECT는 패치 파트 병합 전에도 올바른 결과 반환
--       (병합은 백그라운드에서 진행)
```

---

## LTS vs 일반 릴리스 선택 기준

```
프로덕션 환경
├── 안정성 최우선 → LTS (24.8 또는 25.3)
├── 최신 기능 필요 → 최신 일반 릴리스 (25.12)
└── Kubernetes/매니지드 → 벤더 권장 버전 확인

개발/테스트 환경
└── 최신 릴리스 자유롭게 사용

LTS 업그레이드 주기 권장:
24.3 → 24.8 → 25.3 → 25.8 (6개월마다)
```

---

## 버전 확인 및 업그레이드

```sql
-- 현재 버전 확인
SELECT version();

-- 상세 버전 정보
SELECT * FROM system.build_options WHERE name = 'VERSION_FULL';
```

```bash
# Docker 태그로 버전 지정
docker run clickhouse/clickhouse-server:24.8-lts   # LTS
docker run clickhouse/clickhouse-server:25.3-lts   # 최신 LTS
docker run clickhouse/clickhouse-server:latest      # 최신

# 패키지 업그레이드 (Debian/Ubuntu)
apt-get install clickhouse-server=24.8.*
apt-get install clickhouse-client=24.8.*
```

---

## 출처
- [ClickHouse 2025 연간 정리](https://clickhouse.com/blog/clickhouse-2025-roundup)
- [2025 Changelog](https://clickhouse.com/docs/whats-new/changelog/2025)
- [2024 Changelog](https://clickhouse.com/docs/whats-new/changelog/2024)
- [2023 Changelog](https://clickhouse.com/docs/whats-new/changelog/2023)
- [ClickHouse Release 24.3](https://clickhouse.com/blog/clickhouse-release-24-03)
- [ClickHouse Release 24.8 LTS](https://clickhouse.com/blog/clickhouse-release-24-08)
- [ClickHouse Release 25.10](https://clickhouse.com/blog/clickhouse-release-25-10)
