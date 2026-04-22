# 치트시트 — 빠른 참조

개념 이해가 끝난 뒤에 빠르게 찾아보기 위한 레퍼런스.

## 치트시트 목록

| 파일 | 내용 | 사용 시점 |
|------|------|---------|
| [merge_tree_engines.md](merge_tree_engines.md) | 7가지 MergeTree 엔진 비교, 선택 플로우차트 | 테이블 설계 |
| [type_selection.md](type_selection.md) | 데이터 타입 선택 가이드, 압축 코덱 | 컬럼 정의 |
| [insert_strategy_flowchart.md](insert_strategy_flowchart.md) | INSERT 전략 의사결정 트리 | 수집 파이프라인 구축 |
| [troubleshooting_queries.md](troubleshooting_queries.md) | 증상별 진단 쿼리 모음 | 장애 대응 |
| [version_history.md](version_history.md) | 버전별 주요 변경사항 (23.x ~ 25.x) | 버전 선택/업그레이드 |

## 사용 가이드

### 새 테이블 만들 때

1. `merge_tree_engines.md` → 엔진 선택
2. `type_selection.md` → 타입과 코덱 결정

### 수집 파이프라인 구축할 때

1. `insert_strategy_flowchart.md` → 동기/비동기, 배치 크기
2. `type_selection.md` → 포맷과 압축

### 운영 중 문제 발생 시

1. `troubleshooting_queries.md`에서 증상 찾기
2. 진단 쿼리 바로 실행
