# 3장. Context, Thread Pool, 동시성 제어

ClickHouse 서버는 단일 프로세스 안에서 수많은 작업을 동시에 처리한다. 사용자 쿼리 실행, 백그라운드 머지, 뮤테이션, 레플리카 간 데이터 페치, S3 백업 등이 모두 하나의 서버 프로세스 내에서 벌어진다. 이 장에서는 이 작업들을 조율하는 세 가지 핵심 메커니즘을 다룬다: Context(설정 계층), Thread Pool(스레드 관리), ConcurrencyControl(CPU 자원 경쟁 해소).

---

## 3.1 Context 계층 구조: 설정은 어디서 오는가

2장에서 간략히 언급한 Context를 여기서 깊이 다룬다. ClickHouse의 모든 동작은 설정(settings)에 의해 제어되며, 설정의 출처는 여러 단계에 걸쳐 있다.

### 4가지 Context 유형

| Context | 범위 | 설정 출처 |
|---------|------|-----------|
| **Global context** | 서버 전체 | config.xml / config.yaml 등 설정 파일 |
| **Session context** | 사용자 세션 | profiles, users 설정 + `SET` 커맨드 |
| **Query context** | 개별 쿼리 | `SETTINGS` 절 |
| **Background context** | 백그라운드 작업 | Global + 'background' 프로파일 |

### 설정 머지 순서

서버가 쿼리를 실행할 때 설정을 아래 순서로 머지한다. **뒤의 것이 앞을 오버라이드**한다:

```
① Global defaults (코드 내 하드코딩된 기본값)
  ② Global configuration (설정 파일)
    ③ Profile settings (<profiles> 섹션)
      ④ User settings (<users> 섹션)
        ⑤ Session settings (SET 커맨드)
          ⑥ Query settings (SETTINGS 절)
```

이 순서를 이해하면, "왜 SET으로 바꾼 설정이 특정 쿼리에서 안 먹히는지", "왜 config 파일을 바꿨는데 동작이 안 바뀌는지" 등의 혼란을 피할 수 있다.

### Background Context의 특수성

이것은 운영에서 자주 혼동되는 부분이다. 머지, 뮤테이션 같은 백그라운드 작업은 **Session context와 Query context를 무시**한다. 오직 Global context와 'background' 프로파일만 적용된다.

공식 문서 원문:

> "Background operations can be configured via global and 'background' profile settings; session and query settings have no effect in this case."

따라서 다음과 같은 시나리오가 발생한다:

**잘못된 시도**: `SET max_bytes_to_merge_at_max_space_in_pool = ...` → 세션 설정이므로 백그라운드 머지에 영향 없음

**올바른 방법**: `<profiles>` 섹션의 `background` 프로파일에 해당 설정 추가, 또는 `background_profile` 서버 설정으로 다른 프로파일 지정

---

## 3.2 Thread Pool 계층 구조

ClickHouse는 스레드 생성·소멸 비용을 피하기 위해 **스레드 풀(thread pool)** 체계를 사용한다. 모든 특수 풀은 Global thread pool 위에 구축된 계층 구조를 이룬다.

```
Global Thread Pool (최상위, 싱글톤)
  ├── Server Pool (클라이언트 커넥션)
  ├── IO Thread Pool (I/O 집중 작업)
  │     └── Backups IO Thread Pool (S3 백업 전용)
  ├── Background Schedule Pool (주기적 작업)
  └── MergeTreeBackgroundExecutor (머지/뮤테이션/페치/무브)
```

### Global Thread Pool

모든 풀의 기반이 되는 `GlobalThreadPool` 싱글톤이다. `ThreadFromGlobalPool`로 스레드를 할당받으며, `std::thread`와 유사한 인터페이스를 제공한다.

설정:
- `max_thread_pool_size`: 풀 내 최대 스레드 수
- `max_thread_pool_free_size`: 새 작업을 대기하는 유휴 스레드 수 상한
- `thread_pool_queue_size`: 대기열에 쌓을 수 있는 작업 수 상한

모든 하위 특수 풀은 Global pool에서 스레드를 가져온다. 특수 풀의 역할은 **동시 작업 수를 제한**하고 **작업 스케줄링**을 하는 것이다. 작업이 스레드보다 많으면 우선순위 큐에 쌓이며, 우선순위 값이 높은 작업이 먼저 시작된다 (기본 우선순위는 0). 단, 이미 실행 중인 작업 간에는 우선순위 차이가 없다. 우선순위는 풀이 과부하 상태일 때만 의미가 있다.

### Server Pool

`Poco::ThreadPool` 기반이며, 최대 `max_connection` 개의 스레드를 가진다. **각 스레드가 하나의 활성 커넥션을 전담**한다.

### IO Thread Pool

I/O에 블로킹되는 작업(디스크 읽기/쓰기, 네트워크 대기 등)을 위한 풀이다. 존재 이유가 명확하다:

> "The main purpose of IO thread pool is to avoid exhaustion of the global pool with IO jobs, which could prevent queries from fully utilizing CPU."

I/O 작업이 Global pool의 스레드를 모두 점유하면 CPU 집중 쿼리가 스레드를 할당받지 못하는 상황이 발생한다. IO thread pool은 이를 격리한다.

설정: `max_io_thread_pool_size`, `max_io_thread_pool_free_size`, `io_thread_pool_queue_size`

S3 백업은 I/O가 특히 많으므로, 별도의 `BackupsIOThreadPool`이 존재한다 (`max_backups_io_thread_pool_size` 등). 인터랙티브 쿼리에 대한 백업의 영향을 격리하기 위함이다.

### Background Schedule Pool

`BackgroundSchedulePool`은 주기적 작업 실행을 담당한다. 핵심 보장:

- **동일 작업의 동시 실행 방지**: 한 작업이 실행 중이면 같은 작업이 다시 시작되지 않음
- **실행 시점 제어**: 미래의 특정 시점으로 실행을 미루거나 일시 비활성화 가능

`Context::getSchedulePool()`로 범용 인스턴스에 접근한다.

### MergeTreeBackgroundExecutor

MergeTree 관련 백그라운드 작업(머지, 뮤테이션, 페치, 무브)을 위한 특수 풀이다. 이 풀의 핵심 설계는 **선점 가능한 작업(preemptable tasks)** 지원이다.

`IExecutableTask`를 순서가 있는 **스텝(step)** 시퀀스로 분할할 수 있다. 이 구조의 이점은 짧은 작업이 긴 작업보다 우선 처리되도록 스케줄링할 수 있다는 것이다. 예를 들어 작은 파트의 머지가 큰 파트의 머지보다 먼저 완료되도록 할 수 있다.

`Context::getCommonExecutor()` 등으로 접근한다.

---

## 3.3 ThreadStatus와 Thread Group: 쿼리별 리소스 추적

어떤 풀에서 스레드가 할당되든, 작업 시작 시 `ThreadStatus` 인스턴스가 생성된다. 이 객체는 해당 스레드의 모든 정보를 캡슐화한다:

- Thread ID, Query ID
- Performance counters
- 리소스 소비량

Thread-local 포인터로 관리되어 `CurrentThread::get()` 호출로 어디서든 접근 가능하다. 모든 함수에 매개변수로 전달할 필요가 없다.

### 쿼리 실행과 Thread Group

쿼리 실행 시의 스레드 관계는 다음과 같다:

```
Master Thread (Server pool에서 할당)
  ├── ThreadStatus::QueryScope(query_context) 보유
  ├── ThreadGroupStatus 생성
  │
  ├── Worker Thread 1 → CurrentThread::attachTo(thread_group)
  ├── Worker Thread 2 → CurrentThread::attachTo(thread_group)
  └── Worker Thread N → CurrentThread::attachTo(thread_group)
```

- **Master thread**: 서버 풀의 스레드로, `QueryScope` 객체를 보유하여 쿼리 컨텍스트를 연결
- **Thread group**: Master thread가 생성. 쿼리 실행 중 추가 할당된 모든 스레드가 여기에 연결
- **목적**: `ProfileEvents::Counters`로 프로파일 이벤트를 집계하고, `MemoryTracker`로 쿼리 단위 메모리 소비를 추적

이 구조 덕분에 `system.query_log`에서 쿼리별 메모리 사용량, 읽기/쓰기 바이트, CPU 시간 등을 정확히 확인할 수 있다.

---

## 3.4 ConcurrencyControl: CPU Slot 기반 동시성 제어

### 문제: 동시 쿼리의 CPU 경쟁

`max_threads` 설정의 기본값은 단일 쿼리가 모든 CPU 코어를 활용하도록 설계되어 있다. 그런데 여러 쿼리가 동시에 실행되면?

각 쿼리가 `max_threads`만큼 스레드를 요청하면 OS가 스레드 간 컨텍스트 스위칭으로 공정성을 보장하지만, 이는 **성능 페널티**를 발생시킨다. 스레드가 과도하게 많아지면 캐시 미스, 컨텍스트 스위칭 비용, 메모리 오버헤드가 모두 증가한다.

### 해결: CPU Slot 모델

`ConcurrencyControl`은 **CPU slot** 개념을 도입한다. Slot은 동시성의 단위다:

- 스레드를 실행하려면 먼저 slot을 **획득(acquire)**해야 한다
- 스레드가 중단되면 slot을 **반환(release)**한다
- 서버 전체에서 slot 총 수가 **전역적으로 제한**된다

### Slot 상태 머신

각 slot은 세 가지 상태를 가진 독립적 상태 머신이다:

```
   free ──allocate──→ granted ──acquire──→ acquired
    ↑                                        │
    └──────────── release ←──────────────────┘
```

| 상태 | 설명 |
|------|------|
| `free` | 어떤 쿼리든 할당 가능 |
| `granted` | 특정 쿼리에 할당되었지만, 아직 스레드가 점유하지 않음 (짧은 과도 상태) |
| `acquired` | 특정 쿼리에 할당되고, 실제 스레드가 점유 중 |

### API

ConcurrencyControl의 사용 패턴을 코드로 보면 직관적이다:

**1단계: 리소스 할당**
```cpp
auto slots = ConcurrencyControl::instance().allocate(1, max_threads);
```
최소 1개, 최대 `max_threads`개의 slot을 요청한다. **첫 번째 slot은 즉시 부여**되지만, 나머지는 나중에 부여될 수 있다. 이것이 "soft limit"의 의미다 — 모든 쿼리는 CPU 부하가 높더라도 **최소 1개 스레드를 보장**받는다.

**2단계: 스레드별 slot 획득**
```cpp
while (auto slot = slots->tryAcquire())
    spawnThread([slot = std::move(slot)] { ... });
```
slot이 부여(granted)될 때마다 스레드를 생성한다. 초기에 1개로 시작하더라도 나중에 slot이 추가로 부여되면 스레드가 늘어난다.

**3단계: 전체 slot 수 조정 (런타임)**
```cpp
ConcurrencyControl::setMaxConcurrency(concurrent_threads_soft_limit_num);
```
서버 재시작 없이 런타임에 변경 가능하다.

### 동작 시나리오

64코어 서버에서 `concurrent_threads_soft_limit_num = 64`로 설정한 경우:

**쿼리 1개만 실행 중**: 64개 slot을 모두 획득 가능 → 모든 코어 활용

**4개 쿼리 동시 실행**: 각각 처음에 1개 slot 확보(즉시) → ConcurrencyControl이 공정하게 나머지 60개 slot을 분배 → 각 쿼리 약 16개 스레드로 수렴

**100개 쿼리 동시 실행**: 각각 최소 1개 스레드 보장 (soft limit) → 총 100개 스레드 사용. `concurrent_threads_soft_limit_num`(64)을 초과하지만, 최소 보장 때문에 허용됨. 각 쿼리는 1개 스레드로 시작하고, 다른 쿼리가 끝나면 slot이 해제되어 남은 쿼리가 스케일업

---

## 3.5 전체 그림: 쿼리 하나의 라이프사이클

지금까지의 내용을 종합하여 SELECT 쿼리 하나의 전체 라이프사이클을 추적하면:

```
1. 클라이언트 커넥션 수락
   └── Server pool에서 Master thread 할당

2. Context 구성
   └── Global → Profile → User → Session → Query 순서로 설정 머지

3. 쿼리 파싱 + 해석
   └── Parser → AST → Interpreter → QueryPipeline 생성

4. ConcurrencyControl에서 slot 할당
   └── allocate(1, max_threads) → 최소 1개 slot 즉시 확보

5. 스레드 할당 + 실행
   └── Master thread가 ThreadGroupStatus 생성
   └── 획득된 slot 수만큼 Worker thread 생성 (Global pool에서)
   └── 각 Worker가 thread group에 attach → MemoryTracker, ProfileEvents 집계

6. Pipeline 실행
   └── PipelineExecutor가 Processor DAG 실행
   └── IStorage::read()로 파트에서 데이터 읽기
   └── Block 단위로 필터링, 집계, 정렬 등 처리

7. 결과 출력
   └── Format Processor → WriteBuffer → 클라이언트

8. 정리
   └── slot 반환 → ConcurrencyControl이 대기 중인 다른 쿼리에 재분배
   └── ThreadStatus 소멸, ProfileEvents 기록
   └── system.query_log에 로그 기록
```

---

## 3.6 Key Takeaways

| 항목 | 핵심 내용 |
|------|-----------|
| **Context 계층** | 6단계 설정 머지. Background 작업은 Session/Query 설정 무시 |
| **Thread Pool 계층** | 모든 특수 풀은 Global pool 위에 구축. IO 풀은 CPU 풀 고갈 방지용 격리 |
| **MergeTreeBackgroundExecutor** | 머지/뮤테이션 전담. 선점 가능한 스텝 기반 스케줄링으로 짧은 작업 우선 |
| **ThreadStatus + Thread Group** | 쿼리 단위 메모리·성능 추적의 기반. system.query_log의 데이터 소스 |
| **ConcurrencyControl** | CPU slot 기반. 모든 쿼리에 최소 1스레드 보장(soft limit). 런타임 조정 가능 |
| **Slot 상태 머신** | free → granted → acquired. 점진적 스케일업 지원 |

---

## 3.7 운영 시 주의점

### Thread Pool 관련

- **`max_thread_pool_size` 모니터링**: Global pool이 고갈되면 모든 하위 풀이 영향을 받는다. `system.metrics`에서 `GlobalThread`, `GlobalThreadActive` 메트릭을 관찰하라.
- **IO pool 분리의 중요성**: S3 기반 스토리지를 사용한다면 `max_io_thread_pool_size`와 `max_backups_io_thread_pool_size`를 독립적으로 튜닝하라. 백업이 인터랙티브 쿼리의 IO 스레드를 잡아먹을 수 있다.
- **Server pool(`max_connection`)**: 동시 커넥션 수 상한이다. 커넥션 풀을 사용하는 애플리케이션에서는 이 값이 병목이 되지 않도록 확인하라.

### ConcurrencyControl 관련

- **`concurrent_threads_soft_limit_num` 설정**: 기본값은 0(제한 없음)이다. 동시 쿼리가 많은 환경에서는 CPU 코어 수의 1~2배로 설정하는 것을 고려하라. 너무 낮으면 개별 쿼리 성능이 저하되고, 너무 높거나 0이면 컨텍스트 스위칭 비용이 증가한다.
- **Soft limit의 의미**: 모든 쿼리가 최소 1개 스레드를 보장받으므로, 동시 쿼리 수가 slot 총 수를 초과할 수 있다. 이는 의도된 설계다. "쿼리가 실행조차 안 되는" 상황은 발생하지 않는다.
- **런타임 조정**: 서버 재시작 없이 slot 수를 변경할 수 있으므로, 피크 시간대에 동적으로 조정하는 운영 패턴이 가능하다.

### Background 설정 관련

- **Background 프로파일 확인**: 머지가 느리거나 뮤테이션이 밀리면 `background` 프로파일의 설정을 확인하라. `SET`으로는 변경되지 않는다.
- **`background_profile` 오버라이드**: 기본 `'background'` 대신 커스텀 프로파일을 지정하여, 프로덕션과 백그라운드 작업의 리소스를 명시적으로 분리할 수 있다.

### 모니터링 쿼리 예시

```sql
-- 현재 활성 스레드 현황
SELECT metric, value FROM system.metrics
WHERE metric IN ('GlobalThread', 'GlobalThreadActive',
                 'IOThreads', 'IOThreadsActive',
                 'BackgroundMergesAndMutationsPoolTask',
                 'BackgroundSchedulePoolTask');

-- 쿼리별 메모리 사용량 (Thread Group 기반 집계)
SELECT query_id, memory_usage, peak_memory_usage,
       ProfileEvents['RealTimeMicroseconds'] AS cpu_time_us
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY peak_memory_usage DESC
LIMIT 10;

-- ConcurrencyControl 영향 확인: 쿼리별 실제 스레드 수
SELECT query_id, Settings['max_threads'] AS requested_threads,
       length(thread_ids) AS actual_threads
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 20;
```

---

*다음 장: 4장. Table Parts — MergeTree 스토리지의 물리적 구조*
