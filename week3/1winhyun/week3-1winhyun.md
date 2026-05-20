# Week 3 과제: AI 서비스 및 Agent 도구 시스템 설계 — **DevPilot**

## 1. 문제 이해 및 설계 범위 확정

### 시나리오

사용자가 AI 코딩 Agent에게 자연어로 작업을 요청하면, AI는 단순 텍스트 응답을 생성하는 것이 아니라 **프로젝트 파일 탐색 → 코드 수정 → 명령 실행 → 테스트 수행 → 결과 검증** 등의 Tool을 순차적으로 호출하며 작업을 수행한다. 사용자는 진행 상황과 로그를 실시간으로 확인할 수 있어야 하며, 작업 도중 연결이 끊겨도 상태가 유지되어야 한다.

> **컨셉: DevPilot** — "Cursor/Claude Code/Devin과 같은 클래스의 AI 코딩 Agent SaaS"를 자체 설계한다고 가정. 사용자는 IDE 확장 또는 웹 콘솔에서 자연어 명령을 내리고, 서버 측 Agent가 Sandbox에서 도구를 실행하며 결과를 스트리밍한다. 사용자 본인의 머신이 아닌 **클라우드 Sandbox에서 실행**되는 점이 핵심이다.

### 설계 범위 (In / Out of Scope)

| 포함 (In Scope) | 제외 (Out of Scope) |
|---|---|
| 사용자 요청 처리 흐름 | LLM 자체 학습 / 파인튜닝 |
| Agent 실행 흐름 (loop) | GPU 인프라, Transformer 구조 |
| Tool Calling 구조 | 벡터 모델 구현 |
| 파일 탐색 / 코드 수정 / 명령 실행 / 결과 검증 | IDE 자체 구현 |
| 스트리밍 응답 (토큰 + 작업 이벤트) | 컨테이너 런타임 자체 구현 (gVisor/Firecracker 등) |
| 장시간 작업 처리 / 상태 관리 / 재연결 | 운영체제 / 완전한 보안 솔루션 개발 |
| Sandbox / 권한 제어 | 자체 LLM 개발 |
| 실패 복구 및 재시도 | 빌링 / 결제 |

### 시스템 구성 전제

- 외부 LLM API(Claude/OpenAI/Bedrock)를 사용한다고 가정. **Tool Calling**을 지원하는 모델만 사용.
  - Tool Calling이란?: (LLM이 텍스트 대신 "이 도구를 이런 인자로 실행하라"는 구조화된 호출을 반환하고, 시스템이 실행해 결과를 다시 LLM에 돌려주는 방식)
  - 왜 필요한가?: LLM은 텍스트만 생성할 뿐 파일을 직접 읽거나 명령을 실행할 수 없다. 본 시스템은 실제로 코드를 수정하고 명령을 실행해야 하므로, 이 간극을 메우는 Tool Calling 없이는 Agent loop(3-1) 자체가 성립하지 않는다.
- Tool 실행용 Sandbox(컨테이너 런타임)는 이미 준비되어 있다고 가정. 본 설계는 **그 위에서 Agent를 어떻게 orchestration 하는가**를 다룬다.
- 사용자는 로그인 상태이며, GitHub OAuth로 본인의 Repository에 접근 권한을 위임했다고 가정.
- 외부 LLM API의 장애·지연·rate limit에서 오는 문제는 본 서비스가 책임진다.

### 기능 요구사항

- 사용자의 자연어 요청을 받아 작업을 시작하고, 진행 상황을 실시간 스트리밍한다.
- Agent는 LLM 추론과 Tool 실행을 반복(loop)하면서 작업을 수행한다.
- 작업 중 연결이 끊겨도 reconnect 시 동일 작업의 상태와 로그를 이어받을 수 있다.
- 한 작업당 Tool 호출 5~20회, 최대 수십 분의 장시간 작업을 처리할 수 있다.
- 작업 실패 시 자동 retry 또는 사용자 확인 후 복구가 가능하다.
- 한 사용자가 보내는 작업 간 / 다른 사용자 간 환경은 격리(Sandbox)된다.
- 위험 명령(`rm -rf /`, 외부 네트워크 임의 접근 등)은 Sandbox 정책 또는 Tool 화이트리스트 단계에서 차단한다.

### 비기능 요구사항 (시간 / 지연 목표)

| 항목 | 목표 |
|---|---|
| 첫 응답 시작 시간 (TTFT) | **3초 이내** |
| 스트리밍 청크 지연 | 평균 **1초 이하** |
| 작업 상태 복구 (reconnect → 마지막 이벤트 수신) | **2초 이내** |
| Tool 실패 자동 retry | 최대 3회, exponential backoff |
| 장시간 작업 처리 한도 | 최대 **30분** (그 이상은 명시적 백그라운드 모드) |
| 동시 실행 작업 수 | 약 **20,000건** |
| 작업 상태 유지 | 서버 재시작 후에도 유지 (DB 영속화) |
| Agent 응답 일관성 | 동일 요청 중복 실행 방지 (idempotency key) |

### 개략적 규모 추정

| 항목 | 수치 |
|---|---|
| MAU / DAU | 약 500,000명 / 약 100,000명 |
| 일일 Agent 작업 수 | 약 **2,000,000건** |
| 작업당 평균 Tool 호출 횟수 | 5~20회 (평균 10회) |
| 일일 Tool 호출 수 | 약 **20,000,000회** |
| 평균 작업 시간 | 30초 ~ 10분 |
| 장시간 작업(>5분) 비율 | 약 10% (약 200,000건/일) |
| 동시 실행 작업 수(피크) | 약 **20,000건** |
| 평균 스트리밍 연결 유지 시간 | 약 2~5분 |
| 피크 시간대 | 평일 업무 시간 (KST 10–12시, 14–18시) |

### 본인이 추가로 둔 가정

- **Sandbox 1개 = 1 작업(=1 user session)**: Sandbox는 작업 시작 시 spin-up, 작업 종료 시 destroy. 캐시·warm pool로 cold start를 흡수(미리 데워 둔 컨테이너 재고(warm pool)와 내용물 캐시로, 컨테이너 부팅 지연(cold start)을 사용자 대기 시간에서 백그라운드로 옮긴다는 뜻).
- 한 사용자가 동시에 여러 작업을 띄울 수 있으나, **Sandbox 수 = 무료 플랜 1개 / 유료 플랜 3개**로 제한.
- LLM은 단일 모델이 아닌 **모델 라우팅**: 의도 분류·요약은 작은 모델(Haiku급), 추론·코드 생성은 큰 모델(Sonnet/Opus급).
- 작업의 **roll-back은 Git 기반**으로 한다 (작업 시작 시 브랜치 자동 생성). 파일 시스템 스냅샷은 별도 운영하지 않는다.

---

## 2. 개략적 설계안 제시 및 동의 구하기

### 핵심 흐름

본 시스템은 세 개의 사이클이 동시에 돌아간다.

**(1) 작업 시작 흐름** — 사용자가 자연어 요청을 보내면 API Gateway가 받아 **작업(Task)을 생성**한다. Task는 DB에 영속 저장되고, 메시지 큐에 enqueue된다. 사용자에게는 즉시 `task_id`와 함께 스트리밍 채널이 열린다.

**(2) Agent 실행 loop (서버 측, 비동기)** — Worker가 Task를 dequeue → 컨텍스트(파일·메모리·이전 결과)를 모아 LLM에 전달 → LLM이 "Tool을 호출하라"고 응답하면 Tool Executor가 Sandbox에서 실행 → 결과를 다시 LLM에 피드백. **이 loop는 LLM이 "끝났다"고 응답할 때까지 반복**된다. 각 단계마다 이벤트가 Event Bus로 발행된다.

**(3) 사용자 측 스트리밍** — 사용자는 SSE 또는 WebSocket으로 Event Bus를 구독한다. **토큰 스트리밍**(LLM이 한 글자씩 뱉는 응답)과 **작업 이벤트 스트리밍**(`[파일 읽는 중]`, `[pytest 실행 중]`)이 같은 채널로 흘러나오되 타입으로 구분된다. 연결이 끊겨도 이벤트는 영속 저장되어 있으므로 reconnect 시 마지막 수신 이벤트 이후를 이어 받는다.

핵심은 **(1)과 (2)를 비동기로 분리**한 것이다. 사용자 요청 처리와 실제 Agent 실행이 다른 라이프사이클을 가져야 장시간 작업·연결 단절·서버 재시작에 대응할 수 있다.

### 개략적 아키텍처 다이어그램

![아키텍처](./architecture.png)

위 다이어그램은 앞서 설명한 **세 개의 사이클**이 실제 컴포넌트에 어떻게 매핑되는지 보여준다.

**① 요청 경로 (위 → 아래, 동기·빠름)** — `Web·IDE 클라이언트` → `API Gateway`(인증·rate limit·Task 생성) → `Task Queue`(Kafka/SQS) → `Agent Worker`. 클라이언트는 여기까지만 동기로 기다리고 즉시 `task_id`를 받은 뒤 손을 뗀다.

**② Agent 실행 (Worker의 fan-out, 비동기·느림)** — `Agent Worker`(Orchestrator·loop)가 세 방향으로 분기한다.

- `LLM Client`(모델 라우팅·재시도) → `LLM API`(Claude/OpenAI) — "다음에 무엇을 할지" 추론
- `Tool·Sandbox Pool`(컨테이너 격리 실행) → `Git Provider`(GitHub/GitLab) — 결정된 Tool을 격리 환경에서 실제 실행
- `Event Bus`(Redis Streams) — 매 단계 이벤트 발행

이 세 가지가 3-1의 Agent loop를 이룬다.

**③ 스트리밍 경로 (아래 → 위, 사용자에게 역류)** — `Event Bus` → `Streaming Gateway`(SSE/WebSocket) → **SSE 스트림** → `Web·IDE 클라이언트`. 요청 경로와 분리된 별도 채널이므로, 연결이 끊겨도 ②는 계속 돈다.

**영속화 (점선)** — `Event Bus`에서 `영속 저장소`(Postgres·S3)로 향하는 점선은 비동기 백업이다. Redis가 떨어지거나 Worker가 죽어도 이 저장소에서 작업·이벤트를 복구한다(3-3).

핵심은 **요청 경로(①)와 스트리밍 경로(③)가 Worker(②)를 사이에 두고 분리**되어 있다는 점이다 — 그래서 사용자 연결이 끊겨도, 서버가 재시작해도 작업이 유지된다.

---

## 3. 상세 설계

### 설계 대상 컴포넌트 사이의 우선순위 정하기

8개 질문 중 본 설계의 **본질에 가까운 4개를 깊게**, 나머지는 보조적으로 다룬다. 우선순위 기준은 "이 시스템이 일반적인 LLM API 호출 서비스와 가장 다른 지점은 어디인가"이다.

| 우선순위 | 항목 | 이유 |
|---|---|---|
| ★★★ | 3-1. Agent 실행 흐름 (loop) | Agent의 정체성 — 단발 추론과의 가장 큰 차이 |
| ★★★ | 3-3. 장시간 작업 처리 | "작업 단위가 분 단위 이상"이라는 본질적 제약 |
| ★★★ | 3-4. 스트리밍 응답 구조 | 토큰 + 작업 이벤트 두 종류를 동시에 다뤄야 함 |
| ★★★ | 3-6. Sandbox / 권한 제어 | LLM에게 명령 실행 권한을 주는 시스템의 안전 마지노선 |
| ★★ | 3-2. Tool Calling 구조 | 3-1에 자연스럽게 따라옴 |
| ★★ | 3-7. 실패 복구 / 재시도 | 3-3과 같이 다룸 |
| ★ | 3-5. 컨텍스트 관리 | 본 설계는 외부 LLM 가정, 책의 절대값보다 정책에 집중 |
| ★ | 3-8. 대규모 동시 작업 | 일반 마이크로서비스 스케일링과 유사, 짧게 |

---

### 3-1. Agent 실행 흐름 관리

#### Agent loop의 본질

핵심은 "**LLM 추론 한 번으로는 작업이 끝나지 않는다**"는 점이다. LLM은 매번 "다음에 무엇을 할지"만 결정하고, Worker가 그 결정을 실제로 실행한 뒤 결과를 다시 LLM에 입력한다. 이걸 LLM이 `stop_reason = end_turn` (또는 `max_iterations` 한도 도달) 응답을 줄 때까지 반복한다.

![Agent loop](./AgentLoop.png)

위 그림은 이 loop가 도는 한 사이클을 나타낸다.

- **Task 시작 (Worker pickup)** — Worker가 큐에서 Task를 집어 loop를 연다.
- **LLM 추론 (Tool 결정)** — 컨텍스트를 받은 LLM이 "다음에 어떤 Tool을 어떤 인자로 부를지"만 정한다. 직접 실행하지는 않는다.
- **Tool 실행 (Sandbox 안에서)** — Worker가 그 결정을 격리된 Sandbox에서 실제로 실행한다.
- **점선 = Tool 결과 피드백** — 실행 결과를 다시 `LLM 추론`의 입력으로 되돌린다. 이 되돌림이 loop의 핵심이며, LLM이 `end_turn`(할 일 끝, 정상 종료)을 줄 때까지 반복된다.
- **완료 응답 (end_turn 시그널)** — LLM이 "끝났다"고 응답하면 loop를 빠져나온다.

즉 실선(→)은 진행, **점선은 결과를 다시 먹이는 피드백**이다. 이 피드백 고리가 단발 LLM 호출과 Agent를 가르는 지점이며, loop가 무한히 돌지 않도록 거는 안전장치는 아래 *종료 조건*에서 다룬다.

#### Stateless vs Stateful 선택

**Worker는 stateless, Task 자체는 stateful**로 분리한다. Worker 프로세스가 메모리에 작업 상태를 들고 있으면 worker가 죽거나 재배포될 때 모든 작업이 사라진다. 그래서 매 loop iteration(Agent loop의 한 회전(LLM 추론+Tool 실행 1쌍))마다 다음 3가지를 영속 저장한다:

- **대화 히스토리(messages)**: 시스템 프롬프트 + user 메시지 + 모든 assistant/tool 메시지의 누적 — Postgres 또는 JSONB로 저장
- **Tool 호출 결과**: 큰 결과(파일 내용, 명령 출력)는 Object Storage(S3)에 두고 DB에는 reference만 저장
- **Task 상태**: `queued | running | tool_executing | paused | completed | failed | cancelled`

Worker가 죽으면 다른 Worker가 DB에서 마지막 상태를 읽어 **다음 iteration부터 재개**한다. 단, 이미 실행된 Tool은 다시 실행하지 않도록 idempotency key를 둔다(3-7 참조).

#### 종료 조건

LLM이 무한 loop에 빠지지 않도록 세 가지 안전장치:

1. `stop_reason = end_turn` — 정상 종료
2. `iteration_count > 50` — 강제 종료. 사용자에게 "복잡한 작업이라 50회 단계 내 완료하지 못했습니다" 통지
3. `wall_clock_time > 30분` — 강제 종료. 백그라운드 모드 전환 옵션 제공

---

### 3-2. Tool Calling 구조

#### Tool Registry

**Tool Registry란?** — 시스템이 제공하는 모든 Tool(`read_file`, `run_shell` 등)을 메타데이터와 함께 등록·관리하는 중앙 카탈로그다. Worker는 작업 시작 시 이 Registry에서 "이번 작업·사용자에게 허용된 Tool 목록"을 골라 LLM 프롬프트에 주입하고(3-6의 Tool Whitelist), Tool Executor는 같은 Registry의 정책(timeout·permission·safety_tier)을 근거로 실행을 통제한다. 즉 **LLM이 무엇을 호출할 수 있는지와 시스템이 무엇을 강제하는지를 한곳에서 정의하는 단일 소스**다.

각 Tool은 다음 메타데이터를 가진다. **LLM 프롬프트에 동적으로 주입되는 부분**과 **시스템이 강제하는 부분**이 분리되어야 한다.

| 필드 | 설명 | 예시 |
|---|---|---|
| `name` | LLM이 호출할 때 쓰는 이름 | `read_file`, `run_shell`, `git_diff` |
| `description` | LLM이 보는 사용 설명 (프롬프트에 포함) | "Reads a file from the workspace" |
| `input_schema` | JSON Schema, LLM이 보고 따른다 | `{path: string, max_lines: integer}` |
| `timeout_sec` | 시스템이 강제하는 실행 한도 | `read_file: 10s`, `run_shell: 60s` |
| `retry_policy` | 실패 시 재시도 정책 | `transient_only: 3회, exponential` |
| `permission` | Sandbox 권한 (3-6 참조) | `fs_read`, `fs_write`, `network`, `shell` |
| `safety_tier` | 안전 등급 | `safe`, `requires_confirm`, `blocked` |

핵심은 **LLM이 보는 description과 시스템이 강제하는 정책이 별도**라는 점이다. LLM이 description만 보고 `rm -rf` 같은 시도를 해도, 시스템 단의 `permission` / `safety_tier`에서 차단된다 (3-6).

#### Tool 실행 순서

**LLM이 결정한다.** 즉, Worker는 "다음 Tool을 무엇으로 할지" 직접 결정하지 않는다. 단, LLM이 한 번에 여러 tool_use를 반환할 수도 있으므로 (Claude의 parallel tool use), **병렬 실행 가능 여부는 Tool 메타데이터에 명시**한다.

- `parallel_safe: true` (예: `read_file`, `list_dir`) → 동시 실행
- `parallel_safe: false` (예: `write_file`, `run_shell`) → 순차 실행, 앞 결과를 기다린 후 다음

#### 결과 검증

Tool 결과는 **두 단계**로 검증한다.

1. **구조 검증**: 스키마에 맞게 응답하는지 (`exit_code`, `stdout`, `stderr` 필드 등). 실패하면 즉시 retry.
2. **LLM에게 판정 위임**: 예를 들어 `pytest` 결과의 "성공/실패"는 LLM이 stdout을 읽고 다음 행동을 결정. 시스템은 "잘못된 결과"를 자체적으로 판단하지 않는다 — 이것이 Agent 시스템과 일반 워크플로우의 차이.

---

### 3-3. 장시간 작업 처리

#### Producer-Consumer 분리

API Gateway는 **요청을 받자마자 Task를 DB에 저장하고 큐에 넣은 후 즉시 task_id를 반환**한다. 실제 Agent loop는 별도 Worker pool이 dequeue해서 비동기 실행. 이 분리 덕분에 다음이 가능하다.

- API Gateway는 항상 빠르게 응답 (TTFT 3초 이내 보장에 기여)
- Worker는 작업 단위로 천천히 처리 (수 분~수십 분)
- Worker 인스턴스가 늘어나도 API Gateway는 동일

큐 선택은 **Kafka** (높은 throughput + Event Log 재사용) 또는 **AWS SQS / Google Pub/Sub** (관리 부담 낮음). 본 설계는 **Kafka** 가정 — Event Bus와 Event Log를 동일 인프라(Kafka topic + 영속 저장)로 통합하기 위함.

#### 작업 상태 저장 위치

- **Task 메타데이터** (id, user, status, created_at, updated_at) → **PostgreSQL** (정확한 트랜잭션 필요)
- **Tool 호출 히스토리 + 메시지 누적** → Postgres JSONB 컬럼 (작업당 수십 KB ~ 수 MB)
- **큰 Tool 결과** (파일 내용, 빌드 로그, screenshot) → **S3**, DB에는 reference URL
- **실시간 이벤트 스트림** → **Redis Streams** (TTL 1시간) + Postgres에도 백업

이렇게 두면 Worker가 갑자기 죽어도 DB 한 곳에서 작업 상태 복구가 가능하다.

#### Reconnect 시 상태 복구

스트리밍 연결이 끊겼을 때 클라이언트는 마지막으로 받은 이벤트 ID(`Last-Event-ID` 헤더, SSE 표준)를 보내며 재연결한다. Streaming GW는:

1. Redis Streams에서 `last_event_id` 이후 이벤트를 먼저 replay
2. Redis에 없으면 Postgres의 Event Log에서 가져와 replay
3. 따라잡은 후에는 실시간 publish 구독으로 전환

이 단순한 패턴 덕에 **클라이언트가 30분간 끊겨도 정확히 끊긴 지점부터 이어 받는다**.

#### 서버 장애 시 복구

Worker가 죽으면:

1. **하트비트 감지** — 각 Worker는 5초마다 `worker_id + last_iteration_id`를 Redis에 갱신. 30초간 미갱신 시 dead로 판정.
2. **작업 reclaim** — Supervisor가 dead worker의 모든 `running` Task를 다시 `queued` 상태로 되돌리고 큐에 enqueue (단, 무한 재시도 방지를 위해 `requeue_count`를 증가시키고 임계치 초과 시 `failed`).
3. **재개 위치** — 새 Worker는 DB에서 마지막 iteration을 읽고 다음 iteration부터 재실행. Tool 실행 결과는 이미 DB에 저장되어 있으면 그것을 그대로 사용 (idempotency).

---

### 3-4. 스트리밍 응답 구조

#### 프로토콜 선택: SSE 우선

| 후보 | 본 사례에서의 적합성 |
|---|---|
| HTTP polling | 1초 이하 지연 목표를 만족하기 어려움 ❌ |
| Long polling | 가능하나 연결 관리 비용이 SSE와 비슷 |
| **SSE** | **단방향 서버 → 클라이언트 스트리밍에 최적, HTTP 위 표준, Last-Event-ID 내장** ✓ |
| WebSocket | 양방향이 필요한 경우만 (예: 사용자가 도중에 추가 명령) |

본 시스템은 **사용자 명령 → 응답 스트리밍이 거의 단방향**이므로 SSE를 기본으로 채택. 사용자의 추가 액션(취소·승인)은 별도 REST 엔드포인트로 처리하면 충분하다. ChatGPT, Claude, Anthropic SDK 모두 SSE를 메인 스트리밍 채널로 사용하는 것도 같은 이유.

#### 토큰 스트리밍 vs 작업 이벤트 스트리밍

같은 SSE 채널로 흘려보내되 **이벤트 타입으로 구분**한다.

```
event: token         data: {"text": "안"}
event: token         data: {"text": "녕"}
event: tool_start    data: {"tool": "read_file", "args": {"path": "app.py"}}
event: tool_log      data: {"line": "Reading 142 lines..."}
event: tool_end      data: {"tool": "read_file", "ok": true, "duration_ms": 120}
event: token         data: {"text": "파일을"}
event: task_done     data: {"task_id": "...", "status": "completed"}
```

Tool 실행 로그는 **시작/끝만이 아니라 실시간 라인 단위**로 전달한다. 예를 들어 `pytest` 실행 중 한 줄씩 stdout이 나올 때마다 `tool_log` 이벤트를 발행. 빌드 로그를 보고 싶은 게 사용자 경험의 핵심이기 때문이다.

#### Event Bus 설계

**Event Bus란?** — 컴포넌트 간 직접 호출 대신, "이벤트"라는 메시지를 중간 통로에 발행(publish)하고 관심 있는 다른 컴포넌트가 구독(subscribe)해서 받는 **비동기 메시징 인프라**다. 발신자와 수신자가 서로를 모른 채 동작하게 만드는 **느슨한 결합(decoupling) 계층**이며, 구현체로는 Redis Pub/Sub(비영속·빠름), **Redis Streams(영속·재생 가능)**, Kafka(대용량·장기 보존), RabbitMQ/SQS(워크 큐) 등이 있다.

본 설계에서 Event Bus가 필요한 이유는 세 가지다.

1. **Worker(②Agent 실행)와 Streaming GW(③사용자 전달)의 분리** — Worker는 30분간 Tool을 30회 호출하며 도는데, 사용자 연결은 그 사이에 끊겼다 붙었다 한다. 직접 연결되어 있으면 사용자 연결 끊김이 Worker 작업 실패로 전파된다.
2. **다중 구독자 지원** — 같은 이벤트를 Streaming GW(SSE 전송), Postgres `task_events`(영속 백업), 향후 metrics·audit 등 여러 소비자가 동시에 봐야 한다.
3. **재연결 시 이어 받기** — Redis Streams는 이벤트 ID 기반으로 "이 시점 이후"를 재구독할 수 있다. Pub/Sub였다면 끊긴 사이 이벤트는 영영 받지 못한다.

- **Redis Streams**가 핵심 — pub/sub와 달리 메시지 영속·소비 그룹·재생이 모두 가능
- 각 Task는 자기 stream(`stream:task:{task_id}`)을 가짐
- Streaming GW는 `XREAD BLOCK 1000 STREAMS stream:task:{id} $`로 구독
- 동시에 Worker는 Postgres `task_events` 테이블에도 append (느린 영속화) — Redis가 떨어져도 30분치 이벤트 보존

#### Streaming GW의 역할

**Streaming GW(Streaming Gateway)란?** — 아키텍처 다이어그램에서 `Event Bus`와 `Web·IDE 클라이언트` 사이에 있는 컴포넌트다. Worker가 발행한 이벤트를 Redis Streams에서 구독해(`XREAD`) 사용자에게 SSE/WebSocket으로 흘려보내는 **단방향 전달 계층**이다. Agent 실행(Worker)은 전혀 하지 않고, 오직 "이벤트를 클라이언트에게 밀어주기 + 재연결 시 끊긴 지점부터 이어주기"만 담당한다. 즉 **②Agent 실행과 ③사용자 스트리밍을 분리하는 경계**가 바로 이 컴포넌트다.

Streaming GW는 stateless gateway로, **단일 인스턴스가 단일 Task 연결을 책임지지 않는다**. 즉:

- 사용자가 어느 Streaming GW 인스턴스에 붙어도 Redis Streams에서 동일 Task를 구독 가능
- LB가 sticky session을 강제할 필요가 없다
- Streaming GW가 죽으면 클라이언트가 reconnect → 다른 인스턴스에 붙어도 Last-Event-ID로 정확히 이어 받음

---

### 3-5. 컨텍스트 관리

#### Context window 압박 관리

**Context window란?** — LLM이 한 번의 추론에서 한꺼번에 볼 수 있는 토큰의 최대 길이다. 사람으로 치면 단기 기억 용량에 해당한다. 시스템 프롬프트 + 대화 히스토리 + Tool 호출/결과 + 현재 응답이 **전부 합산**되며, 단위는 **토큰**(영어 1단어 ≈ 1토큰, 한글 1글자 ≈ 1~2토큰)이다. 현재 Claude Sonnet 4.6/Opus 4.7은 200K, Gemini 1.5는 1M~2M, GPT-4o는 128K 수준. 초과 시 API가 거부하거나 오래된 메시지가 잘린다.

에이전트는 LLM이 stateless이므로 **매 턴 전체 히스토리를 통째로 다시 보낸다**. 그래서 Tool 결과(파일 dump, 빌드 로그)가 누적되며 빠르게 부풀고, 한도 근처에서는 "lost in the middle" 현상으로 품질이 떨어지며, 토큰당 과금이라 **비용도 선형 증가**한다. 즉 context window는 **성능·비용·아키텍처를 동시에 결정하는 1차 제약 조건**이다 — OS의 메모리 관리(paging/swapping)와 같은 발상으로, 컨텍스트가 RAM, S3 원본이 디스크, 요약이 압축 페이지에 대응한다.

대부분 모델의 컨텍스트 한도는 200K 토큰 수준이지만, 작업이 30분간 Tool 30회 호출하면 누적 메시지가 한도를 초과할 수 있다. 정책 3단계:

1. **사용 측 압축 (clipping)** — Tool 결과 중 큰 것(파일 전체, 빌드 로그)은 처음 N줄 + 마지막 M줄 + "... (truncated)"로 자른 후 LLM에 입력. 원본은 S3에 보존하고 LLM이 다시 보길 원하면 `read_file_full` Tool로 재조회.
2. **메시지 요약 (summarization)** — 컨텍스트의 75%가 차면 작은 모델(Haiku급)이 오래된 메시지들을 한 단락으로 요약, 원본을 대체. "이전에 user는 X를 요청했고, agent는 A·B·C 파일을 수정했음" 식.
3. **임계치 초과 시 새 Task로 분리** — 정 안 되면 사용자에게 "작업이 너무 커서 분할이 필요합니다"라 알리고 sub-task 생성.

#### Retrieval은 필요한가?

**Retrieval이란?** — LLM에 입력으로 넣을 정보를 외부 저장소(코드베이스·문서·DB)에서 **검색해서 가져오는 행위**다. RAG(Retrieval-Augmented Generation)의 R에 해당한다. LLM은 context window가 유한해 수천 개 파일을 통째로 넣을 수 없고, 학습 데이터에 없는 사내 코드·문서는 알지 못하므로, 질문·작업과 관련 있는 부분만 골라 LLM에 넣어줘야 한다.

방식은 두 가지로 나뉜다.

- **Semantic search (임베딩 기반)** — 사전에 코드/문서를 벡터로 인코딩해 인덱스를 구축하고, 질의를 벡터로 변환해 코사인 유사도 top-K를 반환. Cursor의 codebase indexing, 일반 RAG 챗봇이 이 방식.
- **Agentic search** — LLM이 직접 `grep`, `find`, `read_file` 같은 Tool을 호출하며 스스로 탐색. Claude Code, Devin이 이 방식.

임베딩 방식은 **인덱스 갱신 비용**(커밋마다 변경 파일을 재임베딩)과 **구조적 관계 포착의 한계**(임베딩은 "비슷한 단어"는 잘 잡지만 "이 함수를 호출하는 곳" 같은 호출 그래프는 못 잡음)가 약점이고, agentic search는 LLM이 사람 개발자처럼 entry point → 호출 함수 → ... 식으로 따라갈 수 있고 벡터 DB 운영 부담이 없다는 장점이 있다.

**작업 시작 시점에는 필요하다.** "Redis 연결 오류 원인을 찾아줘"라는 요청에 프로젝트 전체 수천 개 파일을 다 보낼 수 없다. 그래서:

- 작업 초기: 코드베이스 임베딩 인덱스에서 의미상 가까운 파일 top-K를 후보로 노출 (semantic search)
- 또는 LLM이 직접 `grep_tool`, `find_files` 같은 탐색 Tool로 좁혀 가게 함 (agentic search)

본 설계는 **agentic search를 기본**으로 한다 — LLM이 스스로 탐색하면 의도와 더 잘 맞고, 실제로 Claude Code·Devin이 이 패턴을 적극 채택하고 있다(Cursor는 코드베이스 임베딩 인덱스와 agentic 탐색을 병행). 별도 retrieval 인프라를 두면 비용이 크고 인덱스 갱신 지연 문제가 생긴다.

---

### 3-6. Sandbox 및 권한 제어

이 항목은 **AI 코딩 Agent의 안전 마지노선**이다. LLM의 prompt injection이나 잘못된 추론으로도 시스템 전체가 위험에 빠지면 안 된다. **Defense in depth — 한 단계가 뚫려도 다음 단계가 막는다**.

#### 4단 방어선

1. **Tool Whitelist (LLM 단계)** — Worker가 LLM에 노출하는 Tool 목록 자체를 사용자 권한·작업 종류에 따라 동적으로 제한. 예: "코드 리뷰만" 작업에는 `write_file`, `run_shell` Tool을 아예 보여주지 않음 — 호출 자체가 불가능.

2. **Tool Executor 검증 (API 단계)** — LLM이 어떻게든 weird한 인자로 Tool을 호출해도 Tool Executor가 인자를 검증. `read_file(path="../../etc/passwd")` → workspace root 밖이라 reject. `run_shell(cmd="rm -rf /")` → 명령 화이트리스트 위반이라 reject 또는 `requires_confirm` 처리.

3. **Sandbox 격리 (런타임 단계)** — 모든 명령은 **사용자 전용 컨테이너** 안에서 실행. 컨테이너는:
   - 격리된 파일시스템 — workspace volume만 마운트, host filesystem 비노출
   - 격리된 네트워크 — 기본 deny, 명시적 allow list만 (npmjs.org, pypi.org, github.com 등)
   - 자원 제한 — CPU 2core, RAM 4GB, 디스크 10GB, 실행 시간 30분
   - 컨테이너 런타임은 **gVisor 또는 Firecracker** (kernel 격리, 일반 Docker보다 강함)

4. **Audit Log (사후 단계)** — 모든 Tool 호출은 `(user_id, task_id, tool, args, result, timestamp)`로 기록. 위험 패턴(예: `requires_confirm` Tool이 비정상적으로 자주 호출됨, 외부 IP로 outbound 시도)은 모니터링 알람.

#### Sandbox 라이프사이클

| 시점 | 동작 |
|---|---|
| Task 시작 | Sandbox Pool에서 warm 컨테이너 1개 할당 (cold start 회피) |
| 코드 fetch | Git Provider에서 사용자 repo clone, 작업용 브랜치 자동 생성 |
| Tool 호출마다 | 같은 컨테이너에서 실행 (상태 유지) |
| Task 완료 | 변경사항을 PR로 push (사용자 승인 후), 컨테이너 destroy |
| Task 실패/취소 | 컨테이너 즉시 destroy, 작업 브랜치는 30일 보존 (수동 디버깅용) |

**Sandbox는 한 작업당 1개 (격리 단위)**. 같은 사용자라도 다른 작업은 다른 컨테이너에서 — 한 작업의 부작용이 다른 작업으로 새지 않도록.

#### 사용자 승인이 필요한 명령

`safety_tier = requires_confirm`인 Tool은 LLM이 호출하더라도 즉시 실행되지 않는다.

- 예: `run_shell` 중 `rm`, `chmod`, `curl` 같은 위험 명령
- 예: `git push` (외부 시스템 변경)
- 예: `pip install` (의존성 추가)

이런 호출은 Task 상태가 `paused`로 전환되고, 사용자에게 SSE로 `event: confirm_required`가 발행된다. 사용자가 승인 API를 호출하면 재개. **이 패턴은 Claude Code·Cursor의 "permission to edit"와 동일**한 안전 모델이다.

---

### 3-7. 실패 복구 및 재시도

| 실패 유형 | 정책 |
|---|---|
| LLM API timeout / 5xx | exponential backoff (1s → 2s → 4s), 최대 3회 |
| LLM API rate limit | 동일하게 backoff + 다른 provider로 failover (3-8 참조) |
| Tool 실행 timeout | 1회만 retry, 그래도 실패면 LLM에게 "Tool failed: timeout"로 결과 전달 — LLM이 다른 방법 시도 |
| Sandbox crash | 새 Sandbox 할당 + workspace 복원 (Git stash 기반), 작업 재개 |
| Worker crash | 3-3에서 다룬 reclaim 메커니즘 |
| 중복 실행 방지 | 각 Tool 호출에 `idempotency_key = hash(task_id, iteration, tool, args)`. 이미 실행된 키면 저장된 결과 그대로 반환 |

**Rollback**은 일반적으로 시도하지 않는다. Git 기반 작업 브랜치이므로 사용자가 PR 머지 전까지는 main에 영향 없음 — Rollback은 "PR을 머지하지 않거나, 브랜치를 삭제"로 충분하다.

---

### 3-8. 대규모 동시 작업 처리

#### Worker Autoscaling

**Worker Autoscaling이란?** — 작업량(트래픽·큐 적체)에 맞춰 **Agent loop를 도는 Worker 인스턴스 수를 자동으로 늘리고 줄이는 메커니즘**이다. 본 설계의 Worker는 LLM 호출과 Tool 실행을 반복하는 백엔드 프로세스(라인 87, 156)이며, 동시 작업 2만 건(라인 66)을 감당하려면 Worker 수 자체가 트래픽 따라 움직여야 한다.

**왜 autoscaling이 필요한가** — 본 시스템은 트래픽 패턴이 매우 비대칭적이다.

- **시간대별 격차** — 피크 시간(KST 10–12시, 14–18시)과 새벽·주말의 트래픽 차이가 수 배에서 수십 배 (라인 68)
- **고정 운영의 양극단 실패**
  - 피크 기준으로 인스턴스를 고정해두면: 새벽·주말에 대부분이 IDLE → **비용 폭증** (LLM API + 인프라가 가장 비싼 항목)
  - 평소 기준으로 고정해두면: 피크에 큐가 폭주 → **TTFT 3초 목표(라인 47) 깨짐, 30분 대기 발생**
- **스파이크 대응** — 신규 기능 출시·외부 언급·CI 트리거 같은 갑작스러운 유입에 사람이 수동으로 대응할 시간이 없다
- **장애 회복** — 일부 Worker가 죽으면 자동으로 빈자리를 채워 SLO 유지
- **장시간 작업의 누적** — 평균 작업 시간이 30초~10분이고 10%는 5분 초과(라인 64-65)이므로, 신규 작업 유입이 잠깐만 늘어도 동시 실행 작업이 빠르게 누적되어 정적 capacity로는 잡기 어려움

요약하면 autoscaling은 **비용 효율과 SLO를 동시에 달성하기 위한 유일한 현실적 수단**이다. 고정 인스턴스로 둘 다 잡으려면 피크의 10배를 항상 켜두는 식이 되어 경제성이 무너진다.

지표로 CPU/메모리를 쓰는 게 전통적이지만, Agent 워크로드는 LLM 응답을 기다리는 I/O wait 시간이 길어 **CPU는 낮은데 작업은 막혀 있는 상태**가 자주 생긴다. 그래서 사용자가 실제로 체감하는 지연과 직결되는 **큐 지표(Kafka consumer lag + 평균 대기 시간)** 를 쓴다.

정책의 핵심은 **비대칭(asymmetric) scaling**이다. scale-out은 1분만 지속돼도 즉시 +20%로 공격적으로 늘리고, scale-in은 5분간 안정돼야 -10%로 보수적으로 줄인다. 인스턴스 부팅 비용과 캐시 손실이 크기 때문에 늘렸다 줄였다 반복(thrashing)을 피해야 한다.

> 참고: Worker Autoscaling은 **백엔드 프로세스 수**를, 아래 Sandbox warm pool은 **Tool 실행용 컨테이너 재고**를 조절한다. 둘은 다른 레이어이므로 피크 시간 직전 양쪽 모두 사전 증설이 필요하다.

- **트리거 지표**: Kafka consumer lag + 평균 큐 대기 시간
- `lag > 100 messages` 또는 `wait > 10초` 지속 1분 → +20% 인스턴스 추가
- 반대 5분간 `lag < 20` → -10% 축소

#### LLM API rate limit 대응

LLM API의 rate limit (예: 분당 10K tokens)은 본 시스템 외부 제약이므로 가장 까다롭다. 3단 대응:

1. **모델 라우팅 (3-5에서 언급)** — Haiku/Sonnet/Opus를 의도별로 라우팅. Sonnet rate limit이 차도 Haiku로 분류 작업은 계속.
2. **Multi-provider failover** — Anthropic + OpenAI + Bedrock(같은 모델 다른 경로) 셋을 등록해 한 곳이 막히면 즉시 다른 곳으로. Tool Calling 스키마는 provider 별로 약간 다르므로 LLM Client가 normalize.
3. **Burst 시 graceful degradation** — 일정 임계 초과 시 신규 작업은 큐에 대기시키고 "지금 트래픽이 많아 30초 대기 중" 메시지 표시. 이미 시작된 작업은 우선순위 유지.

#### Sandbox warm pool

Sandbox cold start가 3~5초 걸리면 TTFT 목표(3초)를 못 맞춘다. 예측 트래픽 기준 **항상 N개의 warm 컨테이너를 대기**시키고, 새 작업 들어오면 즉시 할당 → 비워진 만큼 백그라운드로 보충. 피크 시간대(KST 10시, 14시) 직전 N을 2배로 늘려 두는 사전 워밍업.

---

## 4. 설계 장점

- **producer-consumer 분리**로 사용자 응답 시간(TTFT 3초)과 실제 작업 시간(분 단위)이 독립. 어느 한쪽이 길어져도 다른 쪽이 영향받지 않는다.
- **모든 상태가 영속**(Task DB + Event Log)이라 Worker·Streaming GW·Sandbox 어느 것이 죽어도 작업 복구 가능. 서버 재시작·배포·롤링 업데이트가 사용자에게 거의 보이지 않는다.
- **SSE + Redis Streams + Last-Event-ID** 조합으로 reconnect 시 정확히 끊긴 지점부터 이어 받는다. SSE 기반 스트리밍과 끊긴 지점 이어받기는 ChatGPT·Claude 등 실제 LLM 서비스에서도 쓰이는 검증된 접근이다(내부 메시지 버스 구현 방식은 서비스마다 다를 수 있다).
- **4단 방어선(Whitelist → Executor 검증 → Sandbox → Audit)** 으로 한 단계가 뚫려도 다음 단계가 막는다. LLM의 prompt injection이나 hallucination으로 인한 위험을 시스템적으로 차단.
- **Tool Calling 스키마와 LLM API 추상화**가 분리되어 있어 LLM provider 교체·동시 사용·failover가 자유롭다.
- **Sandbox 1작업 1컨테이너** 정책으로 작업 간 부작용·데이터 누수 차단. 보안 컴플라이언스(SOC2·ISO 27001)와도 자연스럽게 맞는다.

---

## 5. 설계 단점

- **컴포넌트 수가 많다**. API Gateway, Streaming GW, Worker, Tool Executor, Event Bus, Sandbox Pool, 여러 저장소 — 초기 구축·운영 부담이 크고 작은 팀이 따라가기 어렵다.
- **Sandbox 비용**이 가장 큰 비용 항목. 작업 1건당 컨테이너 1개를 수 분~수십 분 점유. 일 200만 건 × 평균 3분 = 6,000,000분 ≈ 100,000시간 ≈ 약 4,200 컨테이너-day(컨테이너당 2 vCPU 기준 약 8,300 vCPU-day) 수준. 이건 LLM API 비용보다 더 클 수 있다.
- **외부 LLM API 의존성**이 절대적. 여러 provider가 동시에 장애를 겪는 상황에서는 failover만으론 부족. 자체 모델 운영은 본 설계 범위 밖이지만 장기적으로는 옵션.
- **Cold start 비용**과 warm pool 유지 비용 사이의 균형이 까다롭다. 트래픽 예측이 빗나가면 빈 warm pool에 돈을 쓰거나, cold start로 TTFT 목표를 깨거나 둘 중 하나.
- **컨텍스트 압축이 lossy**. 요약·clipping은 LLM 정확도에 영향을 줄 수 있고, "왜 이렇게 결정했는지"가 압축 과정에서 사라질 수 있다.
- **사용자 승인 UX의 burden**. `requires_confirm` Tool이 자주 발생하면 사용자가 매번 yes/no 클릭해야 해서 "Agent의 자동화 가치"가 떨어진다. 사용자별 자동 승인 정책이 필요한데, 이건 또 다른 보안 위험.

---

## 6. 마무리

### 개인적 의견
사실 내가 하지 않았다. AI 설계를 AI에게 맡겼다. 왜냐면 진짜 어떻게 설계를 해야할지 전혀 감이 안잡혔다.
그래서 AI에게 예시를 요청하고 그걸 참고해서 내가 직접 설계를 해볼라고 했는데 예시를 받았을 때 정말 모르는 내용이 천지였다.
그래서 내가 직접 0 to 100을 설계하는건 포기하고 최대한 관련 개념 및 내용들을 학습하며 관련 설계를 짜집기하면서 만들어간 느낌이 강하다.
내가 여태까지 한 개발과 공부한 개념 그 이상을 학습하는 느낌이라 과연 이러한 내용을 전부 실제로 사용할 일이 있을까?라고 생각하면 솔직히 아닌 것 같긴하다.
그치만 새로운걸 배우고 시도하면서 의미있는 학습이 들어오는 느낌을 받았다. 
예를 들면 스트리밍 응답 관련 설계에 있어 조금 더 깊은 학습을 했던 느낌이 있다. 실시간 처리를 진행할 때 단방향, 양방향, 폴링 방식 등에 대해 알고는 있지만 자세히는 알고 있지 않았다. 그치만 이번에 설계를 진행하며 학습을 하면서 각 방식에 대해 조금 더 깊이 학습한 그런거?? 등이 있었다.

이번 스터디를 진행하면서 조금 더 설계를 하는 방식과 시야를 넓혀야겠다는 생각이 들고있다. 내가 여태까지 하던 작은 사이드 프로젝트 그 이상을 설계하기 위한 노력....? 뭐 그런거....

### 사례 공유

- **Anthropic의 Claude Code** — Tool Calling + 사용자 승인 UX + Git 기반 작업 흐름의 실제 사례. 본 설계의 많은 패턴이 여기서 차용됨
- **Cognition Labs의 Devin** — 클라우드 Sandbox에서 자율적으로 실행되는 Agent의 본격 상용 사례, "Agent + Sandbox + Browser" 조합
- **Anthropic의 "Building effective agents" 블로그 글** — Agent loop 설계 시 무엇을 단순하게 두고 무엇을 복잡하게 둘지의 가이드. 본 설계에서 컨텍스트 관리·Tool 추상화 정책에 직접 반영
- **OpenAI Cookbook "Function calling" 가이드 + Anthropic Tool Use 문서** — Tool 스키마 표준화의 산업 관행
- **Hashicorp Nomad / Kubernetes Job API** — 장시간 작업 처리 + 재시도 패턴의 일반적 모델

### 추가 학습할 점

- **MCP (Model Context Protocol)** — Anthropic이 제안한 Tool/Context 통신 표준. 사내 Tool 시스템을 MCP 위로 옮기면 외부 모델 교체가 더 쉬워질 가능성
- **Multi-agent orchestration** — 본 설계는 단일 Agent. 큰 작업을 sub-task로 쪼개 여러 Agent가 협업하는 패턴(planner-executor-critic)은 별도 설계 영역
- **자체 모델 추론 인프라** — LLM API 의존을 줄이려면 vLLM·TGI 기반의 사내 추론 서버. 본 설계의 LLM Client 추상화 덕에 교체는 가능하나 GPU 운영 부담 별도
- **Distributed tracing for Agent loops** — 한 작업이 수십 단계에 걸쳐 여러 컴포넌트를 거치므로 OpenTelemetry 기반 trace + Tool 호출 단위 span 설계가 디버깅의 핵심

---

## 📚 참고 자료

- 《가상 면접 사례로 배우는 대규모 시스템 설계 기초》 — 9장(피드의 push/pull), 12장(채팅 시스템의 영속 메시지·재연결), 14장(YouTube의 비동기 처리)의 패턴이 본 설계에 직접 차용됨
- Anthropic, "Building effective agents" — agent design patterns 가이드
- Anthropic Tool Use Documentation — Tool Calling 스키마와 parallel tool use
- Server-Sent Events 표준 (W3C) — Last-Event-ID, retry 메커니즘
- Redis Streams 공식 문서 — `XADD`, `XREAD`, consumer group
- gVisor / Firecracker 백서 — 멀티테넌트 코드 실행 환경의 커널 격리
- 김영주, "AI Agent 개발 완전 가이드 2025" — Tool Calling, ReAct, Multi-Agent, MCP
