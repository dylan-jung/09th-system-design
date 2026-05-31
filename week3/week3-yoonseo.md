# Week 3 과제: AI DevOps Agent 서비스 및 Tool 시스템 설계

---


#### ⒈ 문제 이해 및 설계 범위 확정

**시나리오**

- 사용자가 AI 코딩 Agent에게 자연어로 작업을 요청하면 AI는 단순히 텍스트 응답만 생성하는 것이 아니라 프로젝트 파일 탐색, 코드 수정, 명령 실행, 테스트 수행, 결과 검증 등의 tool들을 호출하며 작업을 수행한다.
    
    예를 들어:
    
    ```xml
    - "Docker 실행 오류 수정해줘"
    - "Redis 연결 실패 원인 분석해줘"
    - "테스트 코드 작성하고 실행해줘"
    - "이 프로젝트 구조 설명해줘"
    ```
    
    와 같은 요청에 대해 AI Agent는 여러 tool들을 순차적으로 호출하며 작업을 진행한다. 사용자는 작업 진행 상황과 로그를 실시간으로 확인할 수 있어야 하며 긴 작업 도중 연결이 끊겨도 상태가 유지되어야 한다.
    
    그 외 시나리오는 자유롭게 구체화해도 좋다.
    
    ```xml
    AI DevOps Agent
    AI Data Analysis Agent
    AI Browser Agent
    AI 문서 자동화 Agent
    AI 코드 리뷰 Agent... 등등
    ```
    

## 설계 범위 (In / Out of Scope)

---

| 포함 (In Scope) | 제외 (Out of Scope) |
| --- | --- |
| 사용자 요청 처리 흐름 | LLM 자체 학습 |
| Agent 실행 흐름 | 모델 파인튜닝 |
| Tool Calling 구조 | GPU 인프라 |
| 파일 탐색 / 코드 수정 | Transformer 구조 |
| 명령 실행 / 결과 검증 | 벡터 모델 구현 |
| 스트리밍 응답 | IDE 자체 구현 |
| 장시간 작업 처리 | 실제 컨테이너 런타임 구현 |
| 작업 상태 관리 | 운영체제 구현 |
| Sandbox / 권한 제어 | 완전한 보안 솔루션 개발 |
| 실패 복구 및 재시도 | 자체 LLM 개발 |

## 시스템 구성 전제

---

- 외부 LLM API(OpenAI, Claude 등)를 사용한다고 가정
- Tool 실행용 Sandbox(Container)는 이미 준비되어 있다고 가정
- 사용자는 로그인 상태라고 가정
- 파일 저장소 및 Git Repository는 외부 시스템 사용 가능
- AI 서비스는 Tool orchestration과 상태 관리를 책임진다

## 기능 요구사항

---

- 사용자의 자연어 요청을 처리할 수 있어야 한다
- AI Agent는 상황에 따라 여러 Tool을 호출할 수 있어야 한다
- Tool 실행 결과를 기반으로 추가 작업을 수행할 수 있어야 한다
- 작업 진행 상황을 사용자에게 실시간 스트리밍할 수 있어야 한다
- 긴 작업을 비동기로 처리할 수 있어야 한다
- 작업 실패 시 재시도 또는 복구가 가능해야 한다
- 여러 사용자의 동시 작업을 처리할 수 있어야 한다
- Tool 실행 권한 범위를 제한할 수 있어야 한다

## 비기능 요구사항

---

| 항목 | 목표 |
| --- | --- |
| 첫 응답 시작 시간 | 3초 이내 |
| 스트리밍 지연 | 평균 1초 이하 |
| Tool 실행 실패 복구 | 자동 재시도 가능 |
| 장시간 작업 처리 | 최대 수십 분 |
| 작업 상태 복구 | 서버 재시작 이후에도 유지 |
| 동시 실행 작업 수 | 수천 개 이상 |
| Agent 응답 일관성 | 동일 작업 중복 실행 방지 |

## 대략적 규모 추정 *(기준값 — 본인 가정으로 변경 가능)*

---

| 항목 | 수치 |
| --- | --- |
| MAU / DAU | 약 500,000명 / 약 100,000명 |
| 일일 Agent 작업 수 | 약 2,000,000건 |
| 평균 Tool 호출 횟수 | 작업당 5~20회 |
| 평균 작업 시간 | 30초 ~ 10분 |
| 장시간 작업 비율 | 약 10% |
| 동시 실행 작업 수 | 약 20,000건 |
| 평균 스트리밍 연결 유지 시간 | 약 2~5분 |
| 피크 시간대 | 평일 업무 시간대 |

**저장소 추정:**

- 작업 이벤트 로그: 2,000,000건/일 × 20 이벤트 × 평균 2KB = **약 80GB/일**
- 작업 상태(Redis): 20,000건 × 평균 10KB = **약 200MB** (TTL 24h)
- 감사 로그(Audit): 장기 보관용 S3 적재 필요

이 규모에서 핵심 병목은 호출 수 자체보다 **Tool 실행 횟수**다. DAU 10만 × 작업당 평균 10회 Tool 호출 = 하루 약 2천만 번의 Tool 실행이 발생하며, 이를 처리하기 위한 비동기 큐와 Worker 구조가 필수다.

---

## 2. 개략적 설계안 제시

### 핵심 흐름

```
사용자 요청
    │
    ▼
[API Gateway]  ──→  인증/인가 (JWT + Org 권한)
    │
    ▼
[Agent Orchestrator]  ──→  task_id 즉시 반환 (비동기)
    │
    ▼
[작업 큐]  ──→  Worker가 가져감
    │
    ▼
[Agent 실행 루프]
    ├─ LLM: "어떤 Tool을 쓸지" Reasoning
    ├─ Policy Engine: 권한/위험도 확인
    ├─ Tool Executor: Sandbox에서 실행
    ├─ Tool Validator: 결과 검증
    └─ 결과를 LLM Context에 추가 → 반복

실시간 스트리밍: SSE (Server-Sent Events)
작업 상태: Redis (단기) + PostgreSQL (영구)
```

---

### 개략적 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client (Web/IDE Plugin)                │
│          SSE Stream  ◄──────────────────────────────────        │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS / SSE
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                             │
│              (Rate Limiting / Auth / Routing)                   │
└──────────┬────────────────────────────────────────┬────────────┘
           │                                        │
           ▼                                        ▼
┌──────────────────────┐              ┌─────────────────────────┐
│  Agent Orchestrator  │              │   Streaming Gateway      │
│  (Agent 실행 루프 관리)   │◄────────────►│   (SSE 연결 관리)        │
│                      │              │                         │
│  - 의도 파악         │              │  - 클라이언트 연결 풀    │
│  - Tool 선택/실행    │              │  - 이벤트 팬아웃         │
│  - 상태 체크포인트   │              │  - reconnect 처리        │
└──────┬───────────────┘              └─────────────────────────┘
       │
       ├──────────────────────┐
       │                      │
       ▼                      ▼
┌─────────────┐    ┌──────────────────────────────────────────┐
│  LLM API    │    │           Tool Dispatcher                 │
│  (Claude /  │    │                                          │
│   GPT-4o)   │    │  ┌───────────┐  ┌──────────────────┐   │
│             │    │  │  Tool     │  │  Policy Engine   │   │
│  - Reasoning│    │  │  Registry │  │  (권한/위험도)    │   │
│  - Planning │    │  └───────────┘  └────────┬─────────┘   │
│  - Response │    │                          │              │
└─────────────┘    │               승인필요 시 ▼              │
                   │          ┌─────────────────────────┐    │
                   │          │   Approval Service      │    │
                   │          │   (Human-in-the-loop)   │    │
                   │          └─────────────────────────┘    │
                   │                   │ 승인완료             │
                   │                   ▼                     │
                   │  ┌─────────────────────────────────┐   │
                   │  │      Tool Executor               │   │
                   │  │  (Sandbox Container Pool)        │   │
                   │  └────────────────┬────────────────┘   │
                   │                   │                     │
                   │  ┌────────────────▼────────────────┐   │
                   │  │      Tool Validator              │   │
                   │  │  (결과 검증 / 성공 여부 판정)    │   │
                   │  └─────────────────────────────────┘   │
                   └──────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌────────────┐  ┌────────────┐  ┌──────────────┐
       │   Redis    │  │ PostgreSQL │  │   작업 큐     │
       │            │  │            │  │              │
       │ - 작업상태  │  │ - 작업히스 │  │ - 비동기처리 │
       │ - 세션캐시  │  │ - 감사로그  │  │ - 이벤트 큐  │
       │ - Lock     │  │ - 체크포인트│  │              │
       └────────────┘  └────────────┘  └──────────────┘
```

---

## 2-1. 전체 흐름 상세 예시 — "프로덕션 배포 실패, 롤백해줘"

실제 요청이 시스템을 어떻게 흘러가는지 단계별로 추적한다.

---

**1단계 — 요청 접수**

```
사용자: "프로덕션 배포 실패했어. 원인 찾아서 롤백해줘"
  ↓
API Gateway: POST /agent/tasks 수신
  ↓
DB 저장:   task_id=1234 / status=QUEUED
  ↓
작업 큐:   { "task_id": "1234" } 적재
  ↓
사용자에게 즉시 응답: { "task_id": "1234" }   ← 연결 끊김 (비동기)
```

> 핵심은 요청을 받자마자 task_id만 돌려주고 실제 작업은 큐에 넘긴다는 것이다.
> HTTP 연결을 수십 분간 붙잡고 있을 수 없기 때문이다.

---

**2단계 — Worker 할당**

```
Worker가 task 1234 가져감
  ↓
Redis 저장:
  {
    "task_id": "1234",
    "status": "RUNNING",
    "iteration": 0
  }
```

---

**3단계 — 첫 번째 LLM 호출**

Worker가 LLM에게 아래 정보를 전달한다.

```
[사용자 요청]
"프로덕션 배포 실패했어. 원인 찾아서 롤백해줘"

[사용 가능한 Tool 목록]
1. kubectl_get_deployments
2. kubectl_get_logs
3. kubectl_rollout_history
4. kubectl_rollout_undo
```

LLM 응답:
```json
{
  "thought": "배포 상태부터 확인해야겠다",
  "tool": "kubectl_get_deployments",
  "params": { "namespace": "production" }
}
```

> LLM은 직접 실행하지 않는다. "이 Tool을 써라"고 말할 뿐이다.
> 실행은 Orchestrator가 한다.

---

**4단계 — Policy Engine 확인 후 Tool 실행**

```
Orchestrator가 Tool 명세 수신
  ↓
Policy Engine 확인:
  name: kubectl_get_deployments
  timeout: 30초
  permission: k8s:read        ← 읽기 전용
  risk_level: LOW             ← 위험도 낮음
  requires_approval: false    ← 승인 불필요
  ↓
Tool Executor 호출 → Sandbox에서 kubectl 실행
  ↓
결과:
  api-server   3/5 READY   (2개 CrashLoopBackOff)
```

SSE 이벤트 발행:
```
event: tool_started    → [배포 상태 확인 중...]
event: tool_completed  → [완료]
```

---

**5단계 — Tool Validator로 결과 검증**

```
Tool Executor 실행 완료
  ↓
Tool Validator:
  - exit_code: 0 → 정상
  - 출력 파싱 가능 여부 확인
  - 예상 필드(NAME, READY, STATUS) 존재 여부 확인
  ↓
검증 통과 → 결과를 Context에 추가
```

> 검증 실패 시 (파싱 불가, exit_code != 0 등) → 재시도 또는 LLM에 오류 상황으로 전달

---

**6단계 — 결과를 Context에 추가 후 재추론**

```
[누적된 Context]
user:  "배포 실패했어. 롤백해줘"
tool:  kubectl_get_deployments → "api-server 3/5 READY"

LLM 응답:
  "왜 3/5지? 로그 봐야겠다"
  → kubectl_get_logs 호출
```

실행 결과:
```
OOMKilled: container exceeded memory limit 512Mi
```

Tool Validator: 로그 출력 파싱 정상 → Context 추가

---

**7단계 — 루프 반복**

```
[누적된 Context]
user:  "배포 실패했어. 롤백해줘"
tool:  deployment → 3/5 READY
tool:  log → OOMKilled

LLM: "이전 버전 확인 필요"
  → kubectl_rollout_history 호출

결과:
  revision 1: v1.2.3  (이전 안정 버전)
  revision 2: v1.3.0  (현재, 문제 버전)
```

---

**8단계 — 위험 작업 감지 → Policy Engine → Approval Service**

```
LLM: "롤백해야 한다"
  → kubectl_rollout_undo 선택

Policy Engine 확인:
  risk_level: HIGH
  requires_approval: true   ← 프로덕션 변경

승인 정책은 LLM이 아니라 시스템이 결정한다.
LLM이 "괜찮다"고 판단해도 Policy Engine이 막는다.

  ↓
Approval Service 호출
status: AWAITING_APPROVAL

SSE 이벤트:
  event: approval_required
  data: {
    "action": "kubectl rollout undo api-server",
    "message": "프로덕션 롤백입니다. 승인하시겠습니까?"
  }
```

프론트엔드에 승인 버튼 표시.

---

**9단계 — 사용자 승인 → 실행 재개**

```
사용자: POST /approvals/appr-xyz  { "action": "approve" }
  ↓
status: RUNNING 복귀
  ↓
kubectl rollout undo deployment/api-server --to-revision=1 실행
```

---

**10단계 — 검증 (Tool Validator)**

```
rollout undo 완료
  ↓
Tool Validator가 자동으로 검증 실행:
  kubectl rollout status deployment/api-server 호출
  ↓
결과: "successfully rolled out"
  ↓
추가로 kubectl get deployments 재조회
  ↓
결과: api-server 5/5 READY ✅
  ↓
검증 통과 → LLM에 성공 결과 전달
```

> rollout undo 명령이 성공해도 실제로 pod가 정상 기동됐는지는 별도 확인이 필요하다.
> Tool Validator가 이 검증을 자동으로 수행한다.

---

**11단계 — 최종 답변 생성**

```
LLM: 최종 답변 생성
  "프로덕션 api-server를 v1.3.0 → v1.2.3으로 롤백 완료했습니다.
   원인: 신규 버전의 메모리 사용량이 제한(512Mi)을 초과했습니다 (OOMKilled)."

status: COMPLETED
```

SSE 이벤트:
```
event: token_stream  → "프로덕션 api-server를..."  (글자 단위 스트리밍)
event: task_completed
```

---

**전체 루프 요약**

| 단계 | 주체 | 행동 |
|---|---|---|
| 1 | API Gateway | 요청 받고 task_id 즉시 반환 |
| 2 | Worker | 큐에서 task 가져와 시작 |
| 3 | LLM | 상황 추론 → Tool 선택 |
| 4 | Policy Engine | 권한/위험도 확인 |
| 5 | Tool Executor | Sandbox에서 실행 |
| 6 | Tool Validator | 결과 검증 |
| 7~8 | LLM + Orchestrator | 결과 Context 추가 → 반복 |
| 9 | Policy Engine + Approval Service | 위험 작업 감지 → 승인 요청 |
| 10 | 사용자 | 승인 → 실행 재개 |
| 11 | Tool Validator | 실행 후 상태 검증 |
| 12 | LLM | 최종 답변 생성 → 완료 |

> LLM은 생각만 한다. 실행은 항상 Orchestrator가 한다.
> 승인 여부는 LLM이 아니라 Policy Engine이 결정한다.

---

## 3. 상세 설계

### 컴포넌트 우선순위

| 우선순위 | 컴포넌트 | 이유 |
|---|---|---|
| 🔴 P0 | **Agent 실행 흐름 관리** | 전체 시스템의 핵심. 없으면 아무것도 동작하지 않음 |
| 🔴 P0 | **Tool Calling + Validator** | DevOps Agent의 실질적 가치 창출. 결과 검증 없으면 신뢰 불가 |
| 🟠 P1 | **Policy Engine + Approval Service** | 프로덕션 환경 보호의 핵심 |
| 🟠 P1 | **스트리밍 응답 구조** | 사용자 경험에 직결 |
| 🟡 P2 | **장시간 작업 처리** | 배포/빌드 등 핵심 DevOps 작업 지원 |
| 🟡 P2 | **컨텍스트 관리** | 긴 작업에서 LLM 비용 및 성능 최적화 |
| 🟢 P3 | **실패 복구 및 재시도** | 안정성 확보 |

---

### 3-1. Agent 실행 흐름 관리 (심층 분석)

#### Agent 실행 루프 설계

AI DevOps Agent는 LLM이 현재 상황을 추론하고 다음 행동을 결정하는 루프 구조로 동작한다.

```
Step 1: LLM 추론
  입력: 사용자 요청 + 이전 Tool 결과 + 현재 Context
  처리: LLM에 "다음에 무엇을 해야 하는가?" 질문
  출력: Tool 호출 명세 또는 최종 답변

Step 2: Tool 실행
  입력: Tool 호출 명세
  처리: Policy Engine 확인 → Tool Executor → Sandbox 실행
  출력: Tool 실행 결과

Step 3: 결과 검증
  입력: Tool 실행 결과
  처리: Tool Validator가 결과의 유효성 확인
  출력: 검증된 결과 또는 오류

Step 4: Context 업데이트
  입력: 검증된 결과
  처리: 결과를 LLM Context Message로 변환하여 추가
  출력: 업데이트된 Context

Step 5: 종료 판단
  - 최종 답변이 생성되었는가?
  - 최대 반복 횟수를 초과했는가?
  → 조건 미충족 시 Step 1로 복귀
```

---

#### Stateless vs Stateful Agent — Trade-off

DevOps Agent는 **Stateful 방식**을 채택한다.

| 항목 | Stateless | Stateful (채택) |
|---|---|---|
| 장애 복구 | 처음부터 재실행 | 체크포인트에서 재개 |
| 장시간 작업 | 부적합 | 적합 |
| Token 비용 | 매우 높음 (매번 전체 전달) | 낮음 (증분만 전달) |
| 구현 복잡도 | 낮음 | 높음 |

배포 작업은 30분에 달할 수 있고 중간 결과를 계속 참조해야 하기 때문에 Stateful이 필수다. 단, Worker 장애 시 다른 Worker가 이어받을 수 있도록 **상태를 Redis + PostgreSQL에 체크포인트**로 저장한다.

---

#### 체크포인트 구조

```json
{
  "task_id": "task-1234",
  "iteration": 4,
  "status": "RUNNING",
  "context_messages": [
    {"role": "user", "content": "프로덕션 배포 실패했어. 롤백해줘"},
    {"role": "tool", "content": "api-server 3/5 READY"},
    {"role": "tool", "content": "OOMKilled: exceeded memory limit"}
  ],
  "last_checkpoint_at": "2024-03-15T09:01:05Z",
  "worker_id": "worker-node-7"
}
```

Worker 장애 시, 새 Worker는 이 체크포인트를 로드하여 `iteration: 4`부터 재개한다.

---

### 3-2. Tool Calling 구조

#### Tool Registry 구조

모든 Tool은 중앙 Tool Registry에 등록되며, 스펙, 권한 수준, 위험도를 명세한다.

```yaml
tools:
  - name: kubectl_get_deployments
    description: "Kubernetes 배포 상태 조회"
    risk_level: LOW
    required_permission: "k8s:read"
    requires_approval: false
    timeout_seconds: 30
    validator: "check_kubectl_output"   ← 검증 함수 지정

  - name: kubectl_rollout_undo
    description: "Kubernetes 배포 롤백"
    risk_level: HIGH
    required_permission: "k8s:write:production"
    requires_approval: true
    timeout_seconds: 300
    validator: "check_rollout_status"   ← 롤백 후 상태 확인
```

#### Tool 분류 체계

| Category | 설명 | 예시 | 승인 필요 |
|---|---|---|---|
| 조회 | 읽기 전용 조회 | kubectl get, 로그 조회 | ❌ |
| MODIFY | 상태 변경 | kubectl apply, 설정 변경 | 환경에 따라 |
| EXECUTE | 명령 실행 | 빌드 실행, 테스트 실행 | ✅ |

---

#### Tool Validator — 결과 검증 레이어

Tool 실행 후 결과를 그대로 LLM에 넘기지 않는다. Validator가 먼저 결과를 검증한다.

```
Tool 실행 완료
      │
      ▼
Tool Validator
  ① exit_code 확인 (0이 아니면 오류)
  ② 출력 파싱 가능 여부 확인
  ③ Tool별 커스텀 검증 실행
      │
      ├─ 검증 통과 → 결과를 Context에 추가
      │
      └─ 검증 실패 → 재시도 or LLM에 오류 상황 전달
```

**DevOps 관점에서 검증이 중요한 이유:**

kubectl rollout undo 명령이 exit_code 0으로 완료돼도, 실제로 Pod가 정상 기동됐는지는 별도로 확인해야 한다. `check_rollout_status` Validator는 롤백 명령 직후 자동으로 `kubectl rollout status`와 `kubectl get deployments`를 호출하여 실제 상태를 검증한다.

```
rollout undo 실행 (exit_code: 0)
      ↓
check_rollout_status Validator 실행:
  kubectl rollout status deployment/api-server
    → "successfully rolled out" ✅
  kubectl get deployments
    → 5/5 READY ✅
      ↓
검증 통과 → LLM에 "롤백 및 정상 기동 확인" 전달
```

---

#### Policy Engine — 승인 정책을 시스템이 결정

승인 여부는 LLM이 판단하지 않는다. **Policy Engine이 시스템 정책으로 결정**한다.

```
Tool 실행 요청
      │
      ▼
Policy Engine
  ① Tool Registry 허용 목록 확인
  ② Blocklist 패턴 매칭 (rm -rf, kubectl delete namespace 등)
  ③ 환경별 권한 확인 (prod write 권한 여부)
  ④ risk_level 기반 승인 필요 여부 결정
      │
      ├─ LOW  → 즉시 실행
      ├─ HIGH → Approval Service로 전달
      └─ CRITICAL → 무조건 차단 또는 승인
```

LLM이 아무리 "안전하다"고 판단해도 Policy Engine의 규칙을 우회할 수 없다. 승인 정책은 코드로 강제하는 것이지 LLM에게 부탁하는 것이 아니다.

---

### 3-3. 장시간 작업 처리

#### 비동기 작업 큐 선택 — 왜 이 규모에서 Kafka인가

규모를 보면 판단 근거가 나온다.

- 하루 200만 작업 × 작업당 10회 Tool 호출 = **하루 2천만 건의 이벤트**
- 동시 20,000개 작업이 각자 이벤트를 발행
- 서버 장애 시 이벤트 유실 없이 재처리 필요 (이벤트 영속성)
- 추후 감사/분석을 위해 이벤트 스트림 재생 필요

이 조건에서 Redis Streams나 RabbitMQ는 이벤트 영속성과 Consumer 그룹 관리에 한계가 있다. Kafka는 이벤트를 디스크에 영속 저장하고, Consumer 그룹별 독립적인 오프셋 관리가 가능하다. 다만 Kafka 클러스터 운영 부담이 크므로, 초기에는 Redis Streams로 시작하고 규모에 따라 전환하는 것도 현실적인 선택이다.

```
사용자 요청
    │
    ▼
[Agent Orchestrator] → task_id 즉시 반환 (HTTP 202 Accepted)
    │
    ▼
[작업 큐]
    │
    ├──► [Worker Pool: Normal Tasks]    (단기 작업)
    │
    └──► [Worker Pool: Long Tasks]      (배포, 빌드 등 장시간 작업)
```

#### reconnect 시 상태 복구

```
1. 클라이언트 SSE 연결 끊김
2. 재연결 시 마지막 event_id 포함
   GET /api/v1/tasks/{task_id}/stream
   Headers: Last-Event-ID: evt-1234

3. Streaming Gateway가 Redis에서 미전송 이벤트 조회
4. Last-Event-ID 이후 이벤트를 순서대로 재전송
5. 이미 완료된 작업이면 최종 결과를 즉시 전달
```

---

### 3-4. 스트리밍 응답 구조

#### SSE 채택 이유

| 방식 | 장점 | 단점 | 적합성 |
|---|---|---|---|
| HTTP Polling | 구현 단순 | 지연 높음, 서버 부하 | ❌ |
| WebSocket | 양방향, 지연 낮음 | LB 설정 복잡, 운영 부담 | 🔶 과도함 |
| **SSE** | 단방향, HTTP 위에서 동작, reconnect 내장 | 단방향만 가능 | ✅ |

DevOps Agent 스트리밍은 **서버 → 클라이언트 단방향**이다. 사용자 입력(승인/취소)은 별도 REST API로 처리하면 충분하므로 WebSocket의 양방향 기능이 필요 없다. SSE가 더 단순하고 HTTP 인프라를 그대로 활용할 수 있다.

많은 경우 "실시간 = WebSocket"으로 생각하지만, 데이터 흐름 방향을 먼저 분석하는 것이 중요하다.

#### 이벤트 타입 분류

| 이벤트 타입 | 설명 | 단위 |
|---|---|---|
| `task_started/completed` | 작업 생명주기 | 작업 단위 |
| `agent_thinking` | LLM 추론 중 | 추론 단계 단위 |
| `tool_started/completed` | Tool 실행 시작/완료 | Tool 단위 |
| `tool_log` | Tool 실행 실시간 로그 | 라인 단위 |
| `tool_validated` | Tool 결과 검증 완료 | Tool 단위 |
| `token_stream` | LLM 최종 응답 토큰 | 토큰 단위 |
| `approval_required/granted` | 승인 요청/완료 | 이벤트 단위 |
| `error` | 오류 발생 | 이벤트 단위 |

---

### 3-5. 컨텍스트 관리

긴 작업에서 컨텍스트 오버플로우를 방지하는 것은 비용 문제이자 성능 문제다.

#### 메모리 계층 구조

```
┌──────────────────────────────────────────┐
│           작업 메모리 (단기)               │
│  현재 루프의 최근 N개 메시지        │
│  항상 LLM Context에 포함                  │
└──────────────────────────────────────────┘
              │ 오래된 메시지는 요약 압축
              ▼
┌──────────────────────────────────────────┐
│           요약 메모리 (중기)               │
│  "iteration 1~5 요약: OOM 발견,           │
│   rollback history 확인 완료"             │
│  압축된 형태로 Context에 포함             │
└──────────────────────────────────────────┘
              │ 프로젝트 파일은 별도 검색
              ▼
┌──────────────────────────────────────────┐
│           장기 메모리 (Vector DB)          │
│  프로젝트 파일, 설정 파일, 과거 로그 등   │
│  요청 시에만 검색하여 Context에 추가 (RAG)│
└──────────────────────────────────────────┘
```



---

### 3-6. Sandbox 및 권한 제어

**사용자별 작업 환경 격리:**

```
사용자 A의 작업 → Container A (격리)
사용자 B의 작업 → Container B (격리)

격리 수준:
  - Network: 허용된 엔드포인트만 허용 (Egress 화이트리스트)
  - Filesystem: 해당 사용자의 작업 디렉토리만 마운트
  - CPU/Memory: 작업별 리소스 쿼터 설정
  - 권한: 최소 권한 원칙
```

신뢰는 LLM에게 부탁하는 것이 아니라 컨테이너 격리로 물리적으로 강제한다.

---

### 3-7. 실패 복구 및 재시도

**중복 실행 방지 (Idempotency):**

- 모든 Tool 호출에 `idempotency_key = hash(task_id + iteration + tool_name + params)` 부여
- 동일 key를 중복 처리하지 않도록 처리 기록 저장
- "중복 실행 방지용 idempotency key는 Redis에 TTL 걸고 저장"
**재시도 정책:**

```
1회 실패: 즉시 재시도
2회 실패: 5초 대기 후 재시도
3회 실패: 25초 대기 후 재시도 (Exponential Backoff)
초과 시:  FAILED 상태로 Agent에 반환 → LLM이 대안 판단
```

**작업 Rollback:**

```
kubectl apply 전 → kubectl get -o yaml 로 현재 상태 저장
실패 시 → kubectl apply -f {저장된_yaml} 로 복원
```
```
현재 상태
(v1.2.3)

      │
      ▼

현재 상태 백업
(backup.yaml)

      │
      ▼

새 버전 적용
(v1.3.0)

      │
      ▼

실패?
      │
 ┌────┴────┐
 │         │
No       Yes
 │         │
 ▼         ▼

성공     backup.yaml 적용

           │
           ▼

       원래 상태 복구
```      
---

### 3-8. 대규모 동시 작업 처리

**LLM API Rate Limit 대응:**

LLM API는 분당 호출 한도가 있다. 서버는 늘리면 되지만 LLM API는 외부 의존이기 때문에 이것이 실질적 병목이다.

```
대응 전략:
  - Token Bucket으로 API 호출 속도 제한
  - 여러 API Key를 Pool로 관리하여 분산
  - 트래픽 폭주 시 경량 모델(Haiku 등)로 다운그레이드
  - 큐 대기 중 사용자에게 "처리 대기 중" 안내
```

**Worker Autoscaling:**

Worker 수는 Kafka Consumer Lag을 기준으로 HPA를 통해 동적으로 조절한다. Lag이 증가하면 Worker를 늘리고, 줄어들면 감소시킨다. 고정 숫자를 정하기보다 부하에 따라 자동 조정하는 구조가 운영상 현실적이다.

---

## 4. 설계 장점

**① 체크포인트 기반 고가용성**
서버 장애 시 체크포인트에서 정확히 중단된 지점부터 재개한다. 30분짜리 배포 작업이 25분 진행된 시점에 서버가 죽어도 처음부터 다시 하지 않는다.

**② Policy Engine으로 승인 정책 분리**
승인 여부가 LLM의 판단이 아니라 시스템 정책으로 강제된다. LLM이 아무리 자신감 있어도 Policy Engine의 규칙을 우회할 수 없다.

**③ Tool Validator로 결과 신뢰성 확보**
Tool 실행 후 결과를 그대로 믿지 않는다. 검증 레이어를 통과한 결과만 LLM Context에 추가되므로, 잘못된 결과 기반으로 다음 행동을 결정하는 위험을 줄인다.

**④ SSE의 단순성과 안정성**
단방향 스트리밍에 WebSocket 대신 SSE를 선택하여 구현과 운영이 단순하다. HTTP 인프라를 그대로 활용하고 reconnect도 내장 지원한다.

**⑤ Tool Registry 중앙화**
새 Tool 추가나 권한 변경을 코드 수정 없이 Registry 설정으로 처리한다.

---

## 5. 설계 단점

**① Stateful 구조의 운영 복잡도**
Redis와 PostgreSQL이 단일 장애 지점이 될 수 있다. Redis Cluster와 PostgreSQL 복제 구성이 필수이며 운영 부담이 증가한다.

**② LLM API 비용 및 지연**
LLM을 반복 호출하는 구조이기 때문에. 20회 반복 작업은 20번의 API 호출이며, 비용과 누적 지연이 상당하다. 근본적인 한계다.

**③ Human-in-the-loop 병목**
승인 게이트가 강력한 안전장치인 동시에 병목이다. 새벽/휴일 배포 시 사용자가 승인을 못 하면 작업 전체가 멈춘다. 조건부 자동 승인 정책이나 위임 승인 구조가 필요하다.

**④ Tool Validator 구현 부담**
Tool마다 커스텀 검증 함수를 작성해야 한다. Tool 수가 늘어날수록 Validator 유지보수 부담도 증가한다.

**⑤ 컨텍스트 요약의 정보 손실 위험**
요약 과정에서 미묘한 오류 메시지나 경계 조건이 누락될 수 있다. 요약 품질이 Agent 판단에 직접 영향을 준다.

---

## 6. 마무리

### 개인적 의견 / 추가 고려사항

**이 설계에서 가장 중요한 판단 세 가지:**

첫째, 승인 정책을 LLM이 아닌 Policy Engine이 결정하도록 분리한 것이다. AI가 "괜찮다"고 판단해도 프로덕션 변경은 사람의 확인이 필요하다. 이 경계를 코드로 강제하지 않으면 언젠가 사고가 난다.

둘째, Tool Validator를 별도 레이어로 둔 것이다. "명령이 성공했다"와 "원하는 결과가 됐다"는 다르다. rollout undo exit_code 0과 pod 5/5 Ready는 다른 사실이다.

셋째, 컨텍스트 관리를 작업 메모리 / 요약 메모리 / Vector DB 세 계층으로 나눈 것이다. LLM Context에 모든 걸 때려넣으면 토큰 비용이 폭발한다.

**실제 사례:**

Claude Code가 이 설계와 유사한 구조로 동작한다. Tool(bash 실행, 파일 수정 등)을 루프로 호출하며, 위험 작업 전 사용자 확인을 요청한다.

---

### 참고 자료

