# week3 과제
# Week 3 과제: AI 서비스 및 Agent 도구 시스템 설계

## 1. 문제 이해 및 설계 범위

### 시나리오
공개된 CVE, 보안 공지, 패치, 공개 PoC를 수집하고,
Linux 관련 취약점의 원리와 관련 내부 개념을 분석 보고서로 정리하는 Agent이다.  
핵심은 LLM이 직접 실행하지 않고, 서비스가 Tool, 권한, 상태, 컨텍스트를 통제하는 구조를 만드는 것이다.
포함 범위는 사용자 요청 처리, Agent 실행 흐름, Tool Calling, CVE/advisory/패치/PoC 수집, 코드 diff 정적 분석, 작업 상태 관리, Sandbox 권한 제어다.

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

- 자연어 요청을 Agent Run으로 만들 수 있어야 한다.
- Agent는 여러 Tool을 순차적으로 호출할 수 있어야 한다.
- Tool 결과를 바탕으로 다음 분석 단계를 이어갈 수 있어야 한다.
- 긴 작업은 비동기로 처리하고 상태를 복구할 수 있어야 한다.
- 작업 진행 상황을 사용자에게 스트리밍할 수 있어야 한다.
- Tool 실행 권한을 제한하고 위험한 작업을 차단해야 한다.

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

### 규모 추정

- 일일 Agent 작업 수: 약 10건
- 평균 Tool 호출 횟수: 작업당 5~10회
- 평균 작업 시간: 약 10분
- 장시간 작업 비율: 약 10%
- 평균 스트리밍 연결 유지 시간: 약 2~5분

## 2. 개략적 설계안 제시 및 동의 구하기

### 핵심 흐름

사용자는 특정 CVE를 입력하거나, "최근 Linux 취약점 중 공부할 만한 것을 찾아줘"처럼 주제 기반으로 요청할 수 있다.
시스템은 공개 취약점 소스에서 후보를 수집하고, Linux 관련성과 학습 가치를 기준으로 우선순위를 매긴다.
이후 advisory, 패치 diff, 공개 PoC, Linux 개념을 분석해 보고서를 생성한다.
- 사용자 요청은 Agent Run 단위로 관리한다.
- API 서버는 작업을 직접 수행하지 않고 Queue와 Worker에 넘긴다.
- LLM은 Tool 실행을 제안할 뿐 실행 권한은 갖지 않는다.
- Tool Registry와 Policy Engine을 통과한 작업만 실행된다.
- 신뢰할 수 없는 외부 자료는 격리 환경에서 정적 분석한다.
- 모든 단계와 결과는 저장해 상태 복구와 감사가 가능하게 한다.

### 개략적 아키텍처 다이어그램


<img width="1211" height="651" alt="스크린샷 2026-05-20 오후 8 14 40" src="https://github.com/user-attachments/assets/b9e872f8-d772-4729-87f1-492b9730e923" />

- LLM
  - 사용자 의도 해석, 다음 Tool 제안, 중간 요약, 최종 응답 생성
- Agent Orchestrator
  - Agent loop 제어, Tool 호출 순서 관리, 종료 조건 판단
- Tool Registry
  - Tool 목록, schema, timeout, retry, 권한 수준 관리
- Tool Worker
  - 외부 API 조회, 파일 분석, 정적 분석 수행
- Policy Engine
  - Tool 실행 전 권한, 위험도, 승인 필요 여부 판단
- 상태 및 실행 이력 저장소 (Object storage)
  - Run 상태, 응답값, artifact reference
- Sandbox
  - 신뢰할 수 없는 자료를 격리 환경에서 수집하고 정적 분석

## 3. 상세 설계

### 상세 설계 우선순위

이 시스템은 공개 자료와 공개 코드를 다루므로 LLM 판단을 그대로 실행하면 안 된다.
서비스 계층에서 Tool, 권한, 상태, 비용을 통제해야 한다.
특히 prompt injection, 악성 PoC, 과도한 Tool 호출, 중복 실행을 고려한다.

중점 영역은 세 가지다.

1. Tool Calling 구조
2. 컨텍스트 관리
3. Sandbox 및 권한 제어

## 3-2. Tool Calling 구조

- Agent는 LLM의 판단을 사용하지만, LLM이 직접 외부 시스템에 접근하거나 명령을 실행하지 않는다.
- 모든 행동은 Tool Registry에 등록된 Tool을 통해서만 수행한다.
- 각 Tool은 입력 schema, 출력 schema, timeout, retry, permission level을 가진다.
- LLM의 Tool Call 제안 순서
  1. Orchestrator가 schema와 권한 검증
  2. Policy Engine이 실행 가능 여부 판단
  3. Tool Worker 실행
  4. 리즈닝 및 실행 결과를 저장
  5. 다음 LLM 판단에 반영
- Tool 정의 예시
  ```json
  {
    "name": "Tool 이름",
    "description": "LLM에게 노출되는 사용 목적",
    "input_schema": "입력 파라미터 schema",
    "output_schema": "출력 결과 schema",
    "permission_level": "권한 레벨",
    "timeout": "최대 실행 시간",
    "retry_policy": "재시도 가능 여부와 횟수",
    "sandbox_required": "격리 공간 필요 여부",
    "approval_required": "사용자 승인 필요 여부"
  }
  ```

- LLM에 모든 Tool을 노출하지 않는다.
  - 현재 단계에 필요한 Tool의 목록만 노출하고, cache 조회나 event 기록 같은 운영성 Tool은 Orchestrator 내부에서 자동 호출한다.
  - 각 Tool은 최소한의 scope에서 입력/동작을 가져야 한다.
  - `run_command`처럼 범용 실행 권한을 주는 Tool은 없다.

- 주요 Tool 범주는 다음 기준 정도로 생각해 볼 수 있다.
  - CVE / advisory 수집
  - Linux 관련성 및 학습 가치 판단
  - 업스트림의 패치 / source 분석
  - 공개 PoC 후보 검색과 정적 분석
  - Sandbox 격리 수집
  - 지식 그래프 / artifact cache 조회
  - 보고서 생성과 근거 검증
  - 이벤트, quota, 승인 같은 운영 Tool
- Tool 결과는 그대로 신뢰하지 않고 검증한다.
  - CVE ID, 영향 버전, package name은 여러 source로 교차 검증한다.
  - 패치 diff는 upstream commit과 배포판 별 패치를 구분한다.
  - 공개 GitHub repository에 CVE 이름이 붙어 있어도 곧바로 신뢰하지 않고 검증을 우선한다.
    - why? 사용자가 잘못된 PoC를 올리거나, agent 시스템 자체를 공격하기 위한 코드를 올리는 것을 방지하기 위함이다.
  - PoC 신뢰도에 따라서 영속적으로 상태값을 부여하고, 영속적으로 관리 및 참조 가능한지 기록한다.
    - 최종 보고서에는 검증을 통과한 PoC만 근거로 사용한다.

## 3-5. 컨텍스트 관리

- 해당 Agent는 advisory, 취약점 패치 diff, PoC 코드, Linux source, 중간 분석 결과를 다룬다.
- 모든 자료를 LLM 컨텍스트에 넣으면 비용, 지연, 컨텍스트 window, 보안 문제가 생긴다.
- 컨텍스트는 2단계로 나누어 저장한다.
  - 사용자 요청을 포함한 원문은 사용자 영역의 컨텍스트로 유지
    -> 해당 컨텍스트 관리는 LLM에게 의존한다.
  - 공개된 Fact 기반의 조회/분석 결과와 그에 따른 LLM 응답은 공유 영역의 artifact로 저장
    -> 구조화 추출 + Fact 보존 중심으로 Object storage 등에 description 과 함께 저장하여 영속성을 유지한다.


## 3-6. Sandbox 및 권한 제어

- 분석 대상에는 공개된 PoC 및 exploit 코드, 불특정 빌드 스크립트, 외부 저장소의 바이너리가 포함될 수 있다.
- 해당 시스템에서는 PoC 실행 환경을 제공하지 않고 공개된 자료/소스 코드 기반 정적 분석만 수행한다.
  - LLM이 임의 명령을 직접 실행하지 못하게 한다.
  - 모든 실행 입력, 출력, 로그, artifact를 추적 가능하게 저장한다.
- 사전 정의된 원천 데이터 저장소로 한정하여 Public Open 되어있는 검증되지 않은 데이터 수집 Scope를 줄일 수 있다. 
- 각 실행 흐름의 권한 레벨은 다음과 같이 구분해볼 수 있다.
  0. 취약점 공식 메타데이터 조회 (Read-Only)
  1. 공개된 upstream 소스 코드 조회 (Read-Only)
  2. 공개된 PoC/Exploit 코드 조회 및 분석 (실행 불가)
  3. 공개된 소스 코드에 대한 클로닝
  4. ...
- 모든 요청에 대해서 단일 정책 진입점을 둘 수 있다.
  - 단일 정책 진입점(정책 평가)은 Tool 요청 실행 전에 명령, 파라미터, 파일 접근 범위 등을 검사한다.
  - LLM이 자동으로 실행 가능한 명령과 사용자 승인이 필요한 명령을 구분한다. 


## 4. 설계 장점

- LLM이 직접 실행하지 않고 Tool Registry와 Policy Engine을 거치므로 안전하다.
- Tool 책임을 작게 나누어 scope를 줄인다.
- 원문 artifact와 fact 기반의 artifact를 분리하여 공통의 fact를 훼손하지 않고 컨텍스트 효율을 확보한다.
- 공개 PoC를 검증 대상 후보로 다뤄 악성 repository 유입을 줄인다.
- Sandbox와 권한 레벨을 통해 정적 분석 중심의 안전한 실행 바운더리를 둔다.

## 5. 설계 단점

- 모든 자료를 실시간으로 새로 분석하면 지연 시간이 길어질 수 있다.
  - Cold-start, 병렬성에 대한 대응이 되어있지 않다.
- PoC 관련성 검증과 신뢰도 평가는 완전히 자동화하기 어렵다.
- Sandbox 작업과 LLM 호출은 비용 관리가 필요하다.
- 기본적인 동작이 모두 외부 저장소 및 외부 원천 데이터에 의존한다.

## 6. 마무리
- AI Agent는 완전 문외한이라 개념부터 공부하느라 조금 어설프게 되었습니다.
- 이번에 여기저기 물어보면서 ReAct, 멀티턴, 상태 노드 그래프 같은 것들이 있다는 걸 알게 되었네요.
- 이것저것 신경 쓰기보다는 '오직 나만 쓴다.' 정도로 방향을 잡고 짜다 보니, 병렬성이나 아키텍처 관점이 많이 부족했던 것 같습니다.
- 그래도 이번에 좀 새롭게 알게된 것들이 생겨서 실제로 만들어 볼 수 있지 않을까 싶습니다.

참고 자료:
- AI Agent 개발 완전 가이드 2025: Tool Calling, ReAct, Multi-Agent, MCP까지
- https://arxiv.org/abs/2605.07830

