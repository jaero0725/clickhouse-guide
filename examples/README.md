# 실무 도메인 예제

각 예제는 실제 서비스에서 마주치는 요구사항을 기반으로, 지금까지 배운 개념을 어떻게 조합해 해결하는지 보여준다.

## 예제 목록

| # | 도메인 | 다루는 핵심 개념 | 핵심 엔진 |
|---|--------|---------------|---------|
| [01](01_ecommerce_orders.md) | E-commerce 주문/상품 | MergeTree, Replacing, Summing, MV | 복합 |
| [02](02_observability_logs.md) | 애플리케이션 로그 | Async Insert, Map, Bloom filter | MergeTree |
| [03](03_iot_sensors.md) | IoT 센서 시계열 | 코덱, AggregatingMergeTree, 다단계 집계 | Aggregating |
| [04](04_user_behavior_analytics.md) | Product Analytics | HyperLogLog, windowFunnel, Retention | Aggregating |
| [05](05_ad_events.md) | 광고 이벤트 과금 | Collapsing, VersionedCollapsing, 샘플링 | Collapsing |

## 추천 학습 순서

1. **E-commerce** (01): ClickHouse의 가장 흔한 사용 시나리오. 여러 엔진을 조합하는 실제 사례
2. **Observability** (02): Async Insert 실전. 대용량 수집
3. **User Analytics** (04): 복잡한 집계 함수 (HyperLogLog, Funnel)
4. **IoT** (03): 시계열 특화 코덱과 다단계 집계
5. **Ad Events** (05): Collapsing 엔진의 실제 용도

## 공통 원칙

모든 예제에서 반복 등장하는 설계 규칙:

- **ORDER BY 키: 저카디널리티 → 고카디널리티 순서**
- **PARTITION BY: 월/주 단위 (TTL/삭제 목적)**
- **LowCardinality(String)**: 고유값 수천 이하면 무조건
- **Materialized View**: 대시보드 쿼리는 항상 사전 집계
- **Async Insert**: 배칭 불가 시 필수
- **로컬 테이블 직접 INSERT**: Distributed 테이블은 읽기 전용
