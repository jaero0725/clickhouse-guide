# ClickHouse 아키텍처 & 운영 학습 가이드

> 공식 문서 (architecture.md, core-concepts, best-practices) 기반 학습 자료
> 개념 이해 → 실무 예제 → 빠른 참조 3단 구성

---

## 폴더 구조

```
clickhouse-guide/
├── chapters/        ← 개념 학습 (1~11장)
├── examples/        ← 실무 도메인 예제 (E-commerce, IoT 등)
└── cheatsheets/     ← 빠른 참조 (엔진, 타입, 트러블슈팅)
```

---

## 추천 학습 경로

### 🟢 초심자: "ClickHouse가 뭔지 감 잡기"

```
chapters/ch01_clickhouse_overview.md   (왜 쓰는가, 설계 철학)
  ↓
chapters/ch04_table_parts.md           (파트 = 저장의 기본)
  ↓
chapters/ch07_primary_index.md         (성능의 핵심)
  ↓
examples/01_ecommerce_orders.md        (실제 예제로 이해)
```

### 🟡 실무자: "수집 파이프라인 만들기"

```
chapters/ch09_insert_strategy.md
  ↓
cheatsheets/insert_strategy_flowchart.md
  ↓
examples/02_observability_logs.md      (Async Insert 실전)
  ↓
examples/03_iot_sensors.md             (시계열 코덱)
```

### 🟠 고급: "장애 대응과 최적화"

```
chapters/ch10_anti_patterns.md
  ↓
chapters/ch11_monitoring_troubleshooting.md
  ↓
cheatsheets/troubleshooting_queries.md
```

---

## 📚 chapters — 개념 학습

### 1부: 아키텍처 기초

- [**1장. ClickHouse 개요와 설계 철학**](chapters/ch01_clickhouse_overview.md)
  - OLAP vs OLTP, 컬럼 지향 저장, Vectorized Execution
  - **시각화: 행 vs 컬럼 저장 비교, MergeTree vs LSM Tree**

- [**2장. 내부 아키텍처 컴포넌트**](chapters/ch02_internal_components.md)
  - Column, Block, Parser, Interpreter, Processor
  - **시각화: SQL 실행 파이프라인 전 과정**

- [**3장. Context, Thread Pool, 동시성 제어**](chapters/ch03_context_threads_concurrency.md)
  - 설정 계층, CPU Slot 기반 동시성

### 2부: 스토리지 엔진 — MergeTree 깊이

- [**4장. Table Parts**](chapters/ch04_table_parts.md) — 파트의 물리적 구조
- [**5장. Table Partitions**](chapters/ch05_table_partitions.md) — 데이터 lifecycle 관리
  - **시각화: 잘못된 파티셔닝이 파트 폭증을 일으키는 과정**
- [**6장. Table Part Merges**](chapters/ch06_table_part_merges.md) — 백그라운드 머지
  - **시각화: 머지 4단계, FINAL 키워드 성능 저하 원인**
- [**7장. Primary Index**](chapters/ch07_primary_index.md) — Sparse Index, ORDER BY 키 설계
  - **시각화: ORDER BY 키 순서에 따른 850배 성능 차이**

### 3부: 분산 아키텍처

- [**8장. Shards와 Replicas**](chapters/ch08_shards_replicas.md) — 분산 아키텍처
  - **시각화: 분산 쿼리 라우팅, 복제 동작**

### 4부: 운영 실무

- [**9장. INSERT 전략 선택**](chapters/ch09_insert_strategy.md) — 동기/비동기, 배칭
- [**10장. 피해야 할 패턴들**](chapters/ch10_anti_patterns.md) — 안티패턴 모음
- [**11장. 모니터링과 트러블슈팅**](chapters/ch11_monitoring_troubleshooting.md) — 증상별 대응

---

## 🛠️ examples — 실무 도메인 예제

실제 서비스에서 마주치는 요구사항을 ClickHouse로 어떻게 풀어내는지.

| # | 도메인 | 핵심 개념 |
|---|--------|---------|
| [01](examples/01_ecommerce_orders.md) | 🛒 **E-commerce** | 복합 엔진 활용, MV 대시보드 |
| [02](examples/02_observability_logs.md) | 📊 **Observability** | Async Insert, Map, Bloom filter |
| [03](examples/03_iot_sensors.md) | 🌡️ **IoT 시계열** | DoubleDelta/Gorilla 코덱, 다단계 집계 |
| [04](examples/04_user_behavior_analytics.md) | 📈 **Product Analytics** | HyperLogLog, Funnel, Retention |
| [05](examples/05_ad_events.md) | 💰 **광고 이벤트** | Collapsing, 샘플링, 부정 클릭 탐지 |

---

## ⚡ cheatsheets — 빠른 참조

개념을 아는 상태에서 "어떻게 하더라?" 빠르게 찾기 위한 레퍼런스.

| 파일 | 내용 |
|------|------|
| [merge_tree_engines.md](cheatsheets/merge_tree_engines.md) | 7가지 엔진 비교, 선택 플로우차트 |
| [type_selection.md](cheatsheets/type_selection.md) | 타입 선택 가이드, 압축 코덱 |
| [insert_strategy_flowchart.md](cheatsheets/insert_strategy_flowchart.md) | INSERT 전략 의사결정 트리 |
| [troubleshooting_queries.md](cheatsheets/troubleshooting_queries.md) | 증상별 진단 쿼리 모음 |

---

## 핵심 원칙 요약

ClickHouse를 쓸 때 지켜야 할 원칙을 한 페이지에 압축:

### 설계 원칙

```
1. OLTP가 필요하면 ClickHouse를 쓰지 말 것
   → 트랜잭션, 단일 행 UPDATE/DELETE가 잦으면 MySQL/PostgreSQL

2. ORDER BY 키는 "쿼리 WHERE 패턴"에 맞춤
   → 저카디널리티 → 고카디널리티 순서
   → 테이블 생성 후 변경 불가, 사전 설계 필수

3. 파티셔닝은 "데이터 관리용"이지 "쿼리 최적화용" 아님
   → TTL, DROP PARTITION에 사용
   → 파티션 수 1,000 이하 유지 (과도하면 파트 폭증)

4. LowCardinality(String)을 기본값으로
   → 수천 고유값 이하면 무조건 적용

5. 타입은 최소한으로
   → 습관적 Int64/Float64 금지
```

### 수집 원칙

```
1. 배치 INSERT (최소 1만 행) 또는 Async Insert
   → 건 단위 INSERT = "Too many parts" 즉사

2. Distributed 테이블에 INSERT 금지
   → 로컬 테이블에 직접 INSERT

3. UPDATE/DELETE 대신 INSERT-only 패턴
   → ReplacingMergeTree, CollapsingMergeTree 활용
```

### 쿼리 원칙

```
1. 필요한 컬럼만 SELECT
   → SELECT * 는 컬럼 지향의 장점을 죽임

2. 대시보드는 Materialized View로 사전 집계
   → 원본 스캔 금지

3. FINAL 키워드 최소화
   → argMax, sum(sign) 등으로 대체

4. DISTINCT 대신 uniq (HyperLogLog)
   → 정확도 ~1% 희생, 메모리/속도 수백 배 절감
```

---

## 참고 자료

- **공식 Architecture 문서**: [clickhouse.com/docs/development/architecture](https://clickhouse.com/docs/development/architecture)
- **Core Concepts**: parts, partitions, merges, primary-indexes, shards
- **Best Practices**: choosing-a-primary-key, avoid-mutations, avoid-optimize-final, selecting-an-insert-strategy
- **VLDB 2024 Paper**: "ClickHouse - Lightning Fast Analytics for Everyone"
- **ClickHouse SQL Playground**: [sql.clickhouse.com](https://sql.clickhouse.com)
