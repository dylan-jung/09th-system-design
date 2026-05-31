# Week 2 과제: 택시 호출 서비스 설계
## 1. 문제 이해 및 설계 범위 확정

### 시나리오

승객이 호출하면 주변 빈 택시를 찾아 매칭하고, 매칭된 택시가 픽업 지점에 도착할 때까지 승객은 택시 위치와 도착 예상 시간을 실시간으로 확인한다. 픽업 이후 목적지에 도착할 때까지 위치 추적이 이어진다.

본 설계에서는 일반적인 택시 호출 서비스를 가정하면, 서비스 이름은 YunTaxi로 두겠다.
YunTaxi는 단일 대도시권에서 운영되는 실시간 위치 기반 택시 호출 서비스이며, 호출 생성부터 기사 매칭, 픽업 이동, 운행 완료까지의 실시간 흐름에 집중한다.

### 설계 범위 (In / Out of Scope)

| 포함 (In Scope) | 제외 (Out of Scope) |
|---|---|
| 호출 시점부터 도착 완료 시점까지의 실시간 흐름 | 회원가입 · 인증 · 기사 등록 절차 |
| 기사·승객 위치 추적 | 이용 이력 |
| 매칭 로직 | 운행 종료 후 요금 산정 · 결제 · 리뷰 · 정산 |
| 이상 상황 처리 (앱 종료, 네트워크 단절, 취소 등) | 도로망 데이터 / 경로 탐색 알고리즘 |

### 시스템 구성 전제
- 우리가 설계할 서비스는 오로지 택시 호출 서비시인 **YunTaxi** 이다.
- 외부 의존 시스템은 지도 시스템, 회원 시스템, 결제 시스템이 존재한다.
- 지도 시스템은 경로 탐색, 도로 거리, ETA 계산을 제공한다.
- 회원 시스템은 승객/기사 인증 여부와 기본 정보를 제공한다.
- 결제 시스템은 결제 수단 등록 여부 확인 및 운행 후 결제 처리를 담당한다.
- 기사와 승객은 모두 정상적으로 앱에 로그인한 상태이며, 결제 수단도 이미 등록되어 있다고 가정한다.
- 돟로망 데이터 관리, 경로 탐색 알고리즘, 요금 산정, 정산, 리뷰 시스템은 이번 설계 범위에서 제외한다.
- 다만, 외부 시스템 의존으로 인해 발생할 수 있는 지연, 장애, 호출 실패 상황은 YunTaxi 내부에서 완화하거나 fallback 처리한다.

### 기능 요구사항
- 기사 위치를 서버에 저장하고, 승객 호출 시 반경 내 빈 택시를 가까운 순으로 매칭
- 매칭 후 픽업 이동 중: 승객 화면에 택시 위치 + 픽업 지점까지 경로·ETA 실시간 표시
- 운행 중: 승객 화면에 현재 위치 + 목적지까지 경로·ETA 실시간 표시
- 호출·매칭·추적 과정에서 기사 또는 승객 측 앱 종료, 네트워크 단절, 호출 취소 등이 발생해도 시스템은 일관된 호출 상태를 유지해야 한다.

### 비기능 요구사항 (시간 / 지연 목표)

| 항목 | 목표 |
|---|---|
| 호출 접수 응답 시간 | 호출 요청 → "기사 검색 중" 진입까지 **2초 이내** |
| 매칭 완료 시간 | 평균 **30초 이내**, 5분 초과 시 실패 처리 |
| 위치 추적 갱신 지연 | 기사 위치 변화 → 승객 화면 반영 평균 **5초 이내** |

### 개략적 규모 추정 _(기준값 — 본인 가정으로 변경 가능)_

| 항목 | 수치 |
|---|---|
| 서비스 지역 | 단일 대도시권 |
| 누적 가입 승객 | 약 2,000,000명 |
| MAU / DAU | 약 800,000명 / 약 200,000명 |
| 누적 가입 기사 | 약 50,000명 |
| 동시 운행 기사 | 10,000명 |
| 일일 호출 수 | 약 500,000건 |
| 피크 시간 호출 집중도 | 평균 대비 **5배 이상** |
| 피크 시간대 | 평일 출근 07:30–09:30 / 퇴근 18:00–20:00 / 금·토 심야 23:00–02:00 |

### 본인이 추가로 둔 가정
- 서비스는 우선 단일 대도시권에서 운영
- 승객과 기사는 모두 로그인 상태이며, 인증과 결제 수단 등록은 이미 완료된 상태라고 가정한다.
- 승객은 호출 시 출발지와 목적지를 모두 입력한다.
- 호출이 생성되면 승객에게는 먼저 “기사 검색 중” 상태를 빠르게 응답하고, 실제 매칭은 비동기적으로 진행한다.
- 기사는 `AVAILABLE`, `OFFERED`, `PICKUP_IN_PROGRESS`, `ON_TRIP`, `OFFLINE` 중 하나의 상태를 가진다고 가정한다.
- 주변 기사 검색 대상은 모든 기사가 아니라 AVAILABLE 상태의 기사로 제한한다.
- 대기 중인 기사는 약 5초 주기로 위치를 전송하고, 픽업 이동 중 또는 운행 중인 기사는 약 2~3초 주기로 위치를 전송한다고 가정한다.
- 기사 위치는 실시간 검색을 위해 Redis GEO에 저장하고, 호출 상태는 정합성이 중요하므로 PostgreSQL 기반 Ride DB에 저장한다.
- 위치 정보는 최신성이 중요하므로 모든 위치 이력을 영구 저장하지 않고, 실시간 추적에 필요한 최신 위치 중심으로 관리한다.
- 경로와 ETA 계산은 외부 지도 시스템에 의존하되, 호출량과 비용을 줄이기 위해 Route/ETA Service에서 Redis Cache를 활용한다.
- 기사 위치는 자주 갱신되지만, ETA는 매 위치 업데이트마다 재계산하지 않고 10~15초 단위 또는 경로 이탈 시 재계산한다.
- 매칭 방식은 후보 기사에게 순차적으로 한 명씩 제안하는 방식보다, 상위 후보 일부에게 제한적으로 제안하고 먼저 수락한 기사와 매칭하는 방식을 사용한다.
- 여러 기사가 동시에 수락하는 경우에는 Ride DB의 조건부 업데이트를 통해 하나의 기사만 최종 매칭되도록 한다.
- 호출 생성 후 5분 동안 매칭되지 않으면 호출 실패로 처리한다.
- 승객 앱이 종료되거나 네트워크가 끊겨도 호출은 즉시 취소되지 않고, 서버의 Ride 상태를 기준으로 유지된다.
- 기사 앱의 위치 보고가 일정 시간 이상 끊기면 오프라인 또는 이상 상태로 판단하고, 호출 단계에 따라 재매칭 또는 승객 안내를 수행한다.
- 알림 전송은 Kafka 기반 이벤트를 통해 비동기적으로 처리한다.
- 지도 시스템, 회원 시스템, 결제 시스템 장애 시 서비스 전체가 즉시 중단되지 않도록 캐시, fallback, 상태 보류 등의 방식을 고려한다.

## 2. 개략적 설계안 제시 및 동의 구하기
설계 방향

본 설계에서는 택시 호출 시스템을 다음 책임으로 분리한다.

- 호출 상태 관리
- 주변 기사 검색
- 매칭 후보 선정
- 실시간 위치 저장 및 전파
- 경로/ETA 계산
- 알림 전송
- 외부 시스템 연동

핵심 설계 원칙은 다음과 같다.

>위치 데이터와 호출 상태 데이터를 분리한다. 기사 위치는 자주 바뀌고 최신값이 중요하므로 Redis GEO에 저장한다.
반면 호출 상태는 정합성이 중요하므로 PostgreSQL 기반 Ride DB에 저장한다.

또한 외부 지도 시스템 호출은 여러 서비스가 직접 수행하지 않고, `Route/ETA Service`를 통해 격리한다. 이를 통해 지도 API 호출량을 제어하고, 캐싱과 장애 대응을 한 곳에서 처리할 수 있다.

### 개략적 아키텍처
<img width="1061" height="492" alt="week2 개략적 다이어그램 drawio" src="https://github.com/user-attachments/assets/ab04512b-7ba0-4a25-bd4b-105f80e72ec1" />


본 아키텍처는 승객 앱과 기사 앱의 요청을 API Gateway에서 받아 내부 서비스로 라우팅한다.

Ride Service는 호출의 생성, 취소, 매칭 완료, 운행 완료 등 호출 상태의 라이프사이클을 관리한다. 호출 상태는 정합성이 중요하므로 PostgreSQL 기반 Ride DB에 저장한다.

Matching Service는 Ride Service로부터 매칭 요청을 받고, Location Service를 통해 픽업 지점 주변의 가용 기사 후보를 조회한다. 이후 Route/ETA Service를 통해 후보 기사별 ETA를 계산하고, 가장 적합한 기사에게 매칭을 제안한다.

Location Service는 기사 앱으로부터 위치 업데이트를 받아 Redis GEO에 저장한다. Redis GEO는 주변 가용 기사 검색에 사용되는 실시간 위치 인덱스 역할을 한다. 또한 위치 변경 이벤트는 Kafka에 발행되어 Notification Service 또는 Route/ETA Service의 후속 처리에 사용된다.

Route/ETA Service는 외부 지도 시스템을 직접 호출하는 내부 서비스다. 경로와 ETA 결과를 Redis Cache에 짧은 TTL로 저장하여 지도 API 호출량을 줄인다. 지도 시스템 장애나 지연이 발생할 경우에는 마지막으로 계산된 ETA를 사용하거나, 직선거리 기반 fallback을 제공할 수 있다.

Notification Service는 Kafka 이벤트를 구독해 승객과 기사에게 매칭 제안, 매칭 완료, 위치 변경, ETA 변경, 호출 취소 등의 알림을 전달한다.


### 핵심 흐름
#### 1. 호출 생성 흐름

```text
승객 앱 → API Gateway → Ride Service → Ride DB
```
1. 승객이 출발지와 목적지를 입력하고 호출을 요청한다.
2. API Gateway는 요청을 Ride Service로 전달한다.
3. Ride Service는 Ride DB에 호출을 SEARCHING 상태로 저장한다.
4. 승객 앱에는 “기사 검색 중” 상태를 즉시 응답한다.
5. Ride Service는 Matching Service에 매칭 시작을 요청한다.

이 구조를 통해 호출 요청 후 2초 이내에 “기사 검색 중” 상태로 진입할 수 있다.

#### 2. 주변 기사 검색 및 매칭 흐름
```
Ride Service → Matching Service → Location Service → Redis GEO
Matching Service → Route/ETA Service → Redis Cache / 지도 시스템
```
1. Ride Service가 Matching Service에 매칭을 요청한다.
2. Matching Service는 Location Service에 픽업 위치 주변의 가용 기사 후보를 요청한다.
3. Location Service는 Redis GEO에서 반경 내 AVAILABLE 상태의 기사 목록을 조회한다.
4. Matching Service는 후보 기사 목록을 받은 뒤, Route/ETA Service에 후보별 ETA 계산을 요청한다.
5. Route/ETA Service는 먼저 Redis Cache에서 경로/ETA 결과를 조회한다.
6. 캐시에 없으면 외부 지도 시스템을 호출해 경로와 ETA를 계산한다.
7. Matching Service는 ETA, 거리, 기사 상태 등을 기준으로 후보를 정렬한다.
8. 상위 후보 기사에게 매칭 제안을 보낸다.
9. 기사가 수락하면 Ride DB에서 호출 상태를 MATCHED로 변경한다.

#### 3. 실시간 위치 업데이트 흐름

```
기사 앱 → API Gateway → Location Service → Redis GEO
Location Service → Kafka → Notification Service → 승객 앱
```

1. 기사 앱은 주기적으로 현재 위치를 서버에 전송한다.
2. API Gateway는 위치 업데이트 요청을 Location Service로 전달한다.
3. Location Service는 Redis GEO에 기사 최신 위치를 저장한다.
4. Location Service는 DriverLocationUpdated 이벤트를 Kafka에 발행한다.
5. Notification Service는 Kafka 이벤트를 구독한다.
6. 승객 앱에는 기사 위치 변경 정보가 전달된다.

> 이때 기사 위치는 2~5초 단위로 업데이트될 수 있지만, 승객 화면에는 네트워크 부하를 고려해 평균 5초 이내 반영을 목표로 전달한다.

#### 4. ETA 갱신 흐름
```text
Location Service → Kafka → Route/ETA Service → Redis Cache / 지도 시스템
Route/ETA Service → Kafka → Notification Service → 승객 앱
```
1. 기사 위치 변경 이벤트가 발생한다.
2. Route/ETA Service는 ETA 재계산이 필요한지 판단한다.
3. 재계산이 필요하면 Redis Cache에서 최근 경로/ETA를 확인한다.
4. 캐시에 없거나 경로 이탈이 발생한 경우 외부 지도 시스템을 호출한다.
5. 새 ETA가 계산되면 EtaUpdated 이벤트를 Kafka에 발행한다.
6. Notification Service가 승객 앱에 ETA 갱신 정보를 전달한다.

중요한 점은 위치 갱신과 ETA 재계산 주기를 분리한다는 것이다.
```text
기사 위치 업데이트: 2~5초 단위
승객 위치 반영: 평균 5초 이내
ETA 재계산: 10~15초 단위 또는 경로 이탈 시
```

#### 5. 취소 실패 흐름
```text
승객 앱 → API Gateway → Ride Service → Ride DB
Ride Service → Kafka → Notification Service
```

1. 승객이 호출 취소를 요청한다.
2. Ride Service는 현재 Ride 상태를 조회한다.
3. 취소 가능한 상태이면 Ride DB에 CANCELLED 상태로 저장한다.
4. Ride Service는 RideCancelled 이벤트를 Kafka에 발행한다.
5. Notification Service는 기사 앱에 취소 알림을 전송한다.

기사의 위치 보고가 끊긴 경우에는 Location Service가 lastSeenAt을 기준으로 이상 상태를 감지하고 Kafka에 이벤트를 발행한다. 이후 Ride Service 또는 Matching Service가 호출 상태에 따라 재매칭, 실패 처리, 승객 안내를 수행한다.

#### 핵심 컴포넌트 정리

| 컴포넌트                 | 역할                                 | 주요 연동                                         |
| -------------------- | ---------------------------------- | --------------------------------------------- |
| API Gateway          | 앱 요청 진입점, 인증, 라우팅, rate limiting   | 승객 앱, 기사 앱, 내부 서비스                            |
| Ride Service         | 호출 생성, 상태 관리, 취소/완료 처리             | Ride DB, Matching Service, Kafka              |
| Matching Service     | 주변 기사 후보 선정, ETA 기반 정렬, 매칭 제안      | Location Service, Route/ETA Service, Kafka    |
| Location Service     | 기사 위치 저장, 주변 기사 검색, 위치 이벤트 발행      | Redis GEO, Kafka                              |
| Route/ETA Service    | 경로/ETA 계산, 지도 API 호출, 캐싱, fallback | Redis Cache, 지도 시스템, Kafka                    |
| Notification Service | 승객/기사에게 실시간 알림 전달                  | Kafka, 승객 앱, 기사 앱                             |
| Ride DB              | 호출 상태의 기준 저장소                      | PostgreSQL                                    |
| Redis GEO            | 가용 기사 위치 인덱스                       | Location Service                              |
| Redis Cache          | 경로/ETA 캐시                          | Route/ETA Service                             |
| Kafka                | 상태 변경 및 위치 변경 이벤트 전달               | Ride, Matching, Location, Route, Notification |
| 회원 시스템               | 인증 및 사용자 정보 조회                     | Ride Service, API Gateway                     |
| 결제 시스템               | 결제 수단 확인 및 운행 후 결제 처리              | Ride Service                                  |
| 지도 시스템               | 경로 탐색, 도로 거리, ETA 제공               | Route/ETA Service                             |

## 3. 상세 설계

이번 상세 설계에서는 택시 호출 서비스의 핵심인 `주변 가용 기사 검색`을 중심으로 다룬다.

실시간 위치 기반 택시 호출 서비스에서 가장 중요한 것은 많은 기사 위치 업데이트를 효율적으로 처리하면서도, 승객 호출 시점에 주변의 가용 기사를 빠르게 찾는 것이다. 따라서 본 설계에서는 기사 위치 저장 방식과 주변 기사 검색 구조를 상세 설계 대상으로 선정한다.

- 3-2. 기사 위치 저장 — 어디에, 어떤 형태로?

그 외 매칭 대상 선정, ETA 전달, 이상 상황 처리, 지도 API 의존성은 해당 위치 저장 구조와 이벤트 흐름 안에서 필요한 수준으로 함께 다룬다.

---

### 3-2. 기사 위치 저장 — 어디에, 어떤 형태로?

기사 위치는 업데이트 빈도가 높고 최신성이 중요한 데이터이다.  
예를 들어 동시 운행 중인 기사 10,000명이 2~5초마다 위치를 전송한다면, 초당 수천 건의 위치 업데이트가 발생할 수 있다.

이러한 위치 데이터를 모두 PostgreSQL 같은 RDBMS에 저장하면 쓰기 부하가 커지고, 반경 내 기사 검색도 비효율적이다. 따라서 본 설계에서는 기사 위치를 실시간 위치 검색에 적합한 Redis 기반 자료구조에 저장한다.

본 설계에서는 Redis를 다음 두 가지 용도로 나누어 사용한다.

| 저장소 | 자료구조 | 역할 |
|---|---|---|
| Redis GEO | GEO Set | 주변 가용 기사 위치 인덱스 |
| Redis Hash | Hash | 기사 상태, 마지막 위치 보고 시각, 속도, 방향 등 메타데이터 저장 |

핵심 설계 방향은 다음과 같다.

- `Location Service`는 기사 위치 업데이트를 처리하는 **쓰기 경로**를 담당한다.
- `Matching Service`는 주변 기사 후보를 찾는 **읽기 경로**를 담당한다.
- `Health Check Worker`는 위치 보고가 끊긴 기사를 감지하고 Redis GEO에서 제거한다.
- `Kafka`는 위치 변경, 기사 오프라인 감지, 매칭 결과, 호출 상태 변경 이벤트를 전달하는 이벤트 스트림 역할을 한다.
- `Ride Service`는 Kafka 이벤트를 구독해 호출 상태를 Ride DB에 반영한다.
- `Notification Service`는 Kafka 이벤트를 구독해 승객과 기사에게 실시간 알림을 전달한다.

---

### 상세 설계 다이어그램

> 아래 다이어그램은 기사 위치 저장, 주변 기사 검색, 위치 보고 끊김 감지, 이벤트 기반 후속 처리 흐름을 나타낸다.

<img width="1621" height="451" alt="week2 상세 기사위치 drawio" src="https://github.com/user-attachments/assets/086208aa-088d-44bf-8918-0bdc03ba648d" />


```text
기사 앱
  → API Gateway
  → Location Service
      → Redis GEO
      → Redis Hash
      → Kafka

Matching Service
  → Redis GEO
  → Redis Hash
  → Kafka

Health Check Worker
  → Redis Hash
  → Redis GEO
  → Kafka

Kafka
  → Ride Service
  → Notification Service

Ride Service
  → Ride DB
  → Kafka

```

### 핵심 흐름

### 1. 기사 위치 업데이트 흐름

```
기사 앱 → API Gateway → Location Service → Redis GEO / Redis Hash
```

1. 기사 앱은 주기적으로 현재 위치를 서버에 전송한다.
2. API Gateway는 위치 업데이트 요청을 Location Service로 전달한다.
3. Location Service는 기사의 현재 위치를 Redis GEO에 저장한다.
4. Location Service는 Redis Hash에 기사 상태와 메타데이터를 저장한다. 
5. Location Service는 Kafka에 기사 위치 변경 이벤트를 발행한다. 
6. Notification Service는 이 이벤트를 구독해 매칭된 승객에게 기사 위치를 전달한다.

Redis GEO 저장 예시는 다음과 같다.

```
GEOADD geo:drivers:available:seoul {longitude} {latitude} {driverId}
```

Redis Hash 저장 예시는 다음과 같다.
```
HSET driver:status:{driverId}
  status AVAILABLE
  lastSeenAt 2026-05-13T15:30:00
  speed 32
  heading 180
```

Redis GEO에는 모든 기사를 저장하는 것이 아니라, 기본적으로 `AVAILABLE` 상태의 기사만 저장한다. 이미 매칭 제안을 받은 기사, 픽업 이동 중인 기사, 운행 중인 기사, 오프라인 기사는 신규 매칭 후보에서 제외되어야 하기 때문이다.

### 2. 주변 가용 기사 검색 흐름

```
Matching Service → Redis GEO / Redis Hash
```

1. 승객 호출이 발생하면 Matching Service는 픽업 위치를 기준으로 주변 기사 검색을 수행한다.
2. Matching Service는 Redis GEO에 `GEOSEARCH`를 요청해 반경 내 기사 후보를 조회한다.
3. Redis GEO는 반경 내 기사 후보 목록을 반환한다.
4. Matching Service는 Redis Hash에서 각 기사의 상태와 `lastSeenAt`을 확인한다.
5. `AVAILABLE` 상태이고 최근 위치 보고 시각이 충분히 최신인 기사만 최종 후보로 남긴다.
6. 이후 후보 기사 목록은 ETA 계산 및 매칭 제안 단계로 넘어간다.

Redis GEO 검색 예시는 다음과 같다.

```text
GEOSEARCH geo:drivers:available:seoul
    FROMLONLAT {pickupLongitude} {pickupLatitude}
    BYRADIUS 2 km
    ASC
    COUNT 50
```


후보 필터링 기준은 다음과 같이 둘 수 있다.

| 조건     | 기준                            |
| ------ | ----------------------------- |
| 기사 상태  | `AVAILABLE`                   |
| 위치 최신성 | 현재 시각 - `lastSeenAt` ≤ 10~15초 |
| 검색 반경  | 1km → 2km → 3km → 5km 순차 확장   |
| 후보 수   | 최대 50명 내외                     |


이렇게 하면 오래된 위치를 가진 기사나 이미 운행 중인 기사가 매칭 후보에 포함되는 문제를 줄일 수 있다.


### 3. 위치 보고 끊김 감지 흐름

```
Health Check Worker → Redis Hash → Redis GEO → Kafka → Ride Service
```
1. Health Check Worker는 Redis Hash에 저장된 lastSeenAt을 주기적으로 확인한다.
2. 일정 시간 이상 위치 보고가 없는 기사를 오프라인 또는 이상 상태로 판단한다.
3. 해당 기사를 Redis GEO에서 제거한다.
4. Kafka에 위치 보고 끊김 감지 이벤트를 발행한다.
5. Ride Service는 이 이벤트를 구독한다.
6. Ride Service는 Ride DB에서 해당 기사와 연결된 진행 중 호출을 조회한다.
7. 호출 상태에 따라 재매칭, 취소, 실패, 위치 갱신 지연 안내 등을 결정한다.
8. 처리 결과를 다시 Kafka에 호출 상태 변경 이벤트로 발행한다.
9. Notification Service는 해당 이벤트를 구독해 승객 또는 기사에게 알림을 전달한다.

오래된 위치 제거 예시는 다음과 같다.

GEOREM geo:drivers:available:seoul {driverId}

오프라인 판단 기준은 다음과 같이 둘 수 있다.

| 목적        |              기준 예시 | 처리                       |
| --------- | -----------------: | ------------------------ |
| 매칭 후보 필터링 | 10~15초 이상 위치 보고 없음 | 후보에서 제외                  |
| 오프라인 감지   |    30초 이상 위치 보고 없음 | Redis GEO 제거, OFFLINE 처리 |
| 장기 미응답    |    60초 이상 위치 보고 없음 | 호출 상태에 따라 재매칭/실패 처리      |


### 4. 이벤트 기반 후속 처리 흐름

본 설계에서는 위치 변경, 매칭 결과, 기사 오프라인 감지, 호출 상태 변경을 Kafka 이벤트로 전달한다.

| 이벤트             | 발행 주체               | 구독 주체                | 목적                         |
| --------------- | ------------------- | -------------------- | -------------------------- |
| 기사 위치 변경 이벤트    | Location Service    | Notification Service | 승객 화면에 택시 위치 반영            |
| 위치 보고 끊김 감지 이벤트 | Health Check Worker | Ride Service         | 기사 오프라인에 따른 호출 상태 처리       |
| 매칭 결과 이벤트       | Matching Service    | Ride Service         | 매칭 성공/실패 결과를 Ride DB에 반영   |
| 호출 상태 변경 이벤트    | Ride Service        | Notification Service | 승객/기사에게 매칭 완료, 취소, 실패 등 알림 |

이벤트 기반으로 처리하는 이유는 알림, 상태 변경, 위치 전파를 동기 호출로 강하게 묶지 않기 위해서이다.
예를 들어 Notification Service에 일시적인 장애가 발생하더라도, Location Service의 위치 저장이나 Ride Service의 상태 변경 자체가 직접적으로 실패하지 않도록 분리할 수 있다.


---

### 상태별 Redis GEO 등록 정책

Redis GEO는 주변 기사 검색을 위한 위치 인덱스이므로, 신규 매칭 가능한 기사만 포함하는 것이 중요하다.

| 기사 상태                | Redis GEO 등록 여부 | 이유               |
| -------------------- | --------------: | ---------------- |
| `AVAILABLE`          |              등록 | 신규 호출 매칭 가능      |
| `OFFERED`            |     제외 또는 일시 제외 | 이미 매칭 제안을 받은 상태  |
| `PICKUP_IN_PROGRESS` |              제외 | 특정 승객을 픽업하러 이동 중 |
| `ON_TRIP`            |              제외 | 운행 중이므로 신규 호출 불가 |
| `OFFLINE`            |              제외 | 위치 신뢰 불가         |


기사가 운행을 완료하고 다시 대기 상태가 되면 `AVAILABLE` 상태로 전환되며 Redis GEO에 다시 등록된다.

---

### 검색 반경 확장 정책

택시 밀도는 지역에 따라 다르다. 도심에서는 1km 이내에 충분한 기사가 있을 수 있지만, 외곽에서는 반경 내 기사가 없을 수 있다. 따라서 검색 반경은 고정하지 않고 점진적으로 확장한다.

| 단계 | 검색 반경 | 설명           |
| -- | ----: | ------------ |
| 1차 |   1km | 기본 검색 반경     |
| 2차 |   2km | 후보가 없을 경우 확장 |
| 3차 |   3km | 외곽 지역 대응     |
| 4차 |   5km | 최대 검색 반경     |
| 실패 | 5분 초과 | 매칭 실패 처리     |


반경을 무한정 확장하면 승객의 대기 시간이 지나치게 길어지고 기사 도착 ETA도 커진다. 따라서 최대 반경과 최대 매칭 시간을 제한한다.

---

### 위치 업데이트 주기

기사 위치 업데이트 주기는 기사 상태에 따라 다르게 설정한다.

| 기사 상태                | 위치 업데이트 주기 | 설명              |
| -------------------- | ---------: | --------------- |
| `AVAILABLE`          |       약 5초 | 주변 기사 검색 정확도 확보 |
| `OFFERED`            |     약 3~5초 | 제안 중 위치 최신성 유지  |
| `PICKUP_IN_PROGRESS` |     약 2~3초 | 승객이 택시 접근 상황 확인 |
| `ON_TRIP`            |     약 2~5초 | 운행 중 위치 추적      |
| `OFFLINE`            |         없음 | Redis GEO에서 제거  |


위치는 자주 업데이트되지만, 모든 위치 변경을 그대로 승객에게 전달하지는 않는다.
승객 화면에는 네트워크 부하를 고려해 평균 5초 이내 반영을 목표로 한다.

## 설계 장점
### 1. 실시간 기사 검색에 적합
Redis GEO를 사용하면 특정 좌표 기준 반경 내 기사 후보를 빠르게 조회 가능하다.
PostgreSQL에서 위도/경도 조건을 직접 계산하는 방식보다 실시간 위치 검색에 적합하다고 한다.

### 2. 위치 데이터와 호출 상태 데이터 분리
기사 위치는 Redis GEO/Hash에 저장하고, 호출 상태는 PostgreSQL 기반 Ride DB에 저장한다.
```
기사 위치 -> Redis GEO / Redis Hash
호출 상태 -> Ride DB(PostgreSQL)
```

이를 통해 쓰기 빈도가 높은 위치 데이터와 정합성이 중요한 호출 상태 데이터를 각각 성격에 맞는 저장소에 저장할 수 있다.

### 3. 쓰기 경로와 읽기 경로가 분리되어 있다.
위치 서비스는 기사 위치 업데이트를 Redis에 쓰는 역할을 담당하고, 매칭 서비스는 Redis에서 주변 기사 후보를 읽는 역할을 담당한다.
-> 위치 업데이트 처리와 매칭 후보 검색의 로직이 과도하게 의존하지 않는다.

### 4. 오래된 위치 정보로 인한 잘못된 매칭 감소

Redis Hash에 `lastSeenAt`을 저장하고, 매칭 서비스와 헬스 체크 워커가 이를 활용한다.

- 매칭 서비스는 오래된 위치를 가진 기사를 후보에서 제외
- Health Check Worker는 일정 시간 이상 위치 보고가 없는 기사를 Redis GEO에서 제거

이를 통해 실제로 오프라인 상태인 기사가 승객에게 매칭되는 문제 감소

### 5. 알림 서비스 장애가 핵심 상태 처리에 영향 X
알림 서비스는 Kafka 이벤트를 구독해 알림을 전달하기에 알림 서비스에 일시적 장애가 생겨도, 위치 서비스의 위치 저장이나, 탑승 서비스의 호출 상태 저장은 별도 처리 가능

## 설계 단점
### Redis 의존도, Redis GEO와 Hash 간 정합성, 운영 복잡도, 비용 증가
- 주변 기사 검색이 Redis GEO에 의존하여 Redis 장애 시 매칭 품질 떨어짐
- 기사 위치는 Redis GEO에, 기사 상태와 lastSeenAt은 Redis Hash에 저장하기 때문에 두 자료구조가 동시 갱신이 되지 않으면, 아래와 같은 문제 발생 가능
- 위치 업데이트 주기가 짧을수록 서버와 네트워크 부하가 커진다.
- Redis GEO는 직선거리 기반 후보 검색에는 적합하나, 실제 도착 시간은 도로 구조와 교통 상황에 따라 달라질 수 있음.
```text
Redis GEO에는 기사가 존재하지만 Redis Hash 상태는 OFFLINE
Redis Hash는 AVAILABLE인데 Redis GEO에는 위치 없음
```

그래서 위치 서비스에서 위치와 상태 갱신을 하나의 처리 흐름으로 묶고, Health Check Worker가 주기적으로 비정상 데이터를 정리해야 함.

## 마무리

### 설계 요약

본 설계는 택시 호출 서비스에서 가장 중요한 `주변 가용 기사 검색`을 중심으로 상세 설계를 진행했다.

핵심은 다음과 같다.

```text
Location Service는 기사 위치 업데이트를 Redis에 저장한다.
Matching Service는 Redis GEO와 Redis Hash를 읽어 주변 가용 기사 후보를 찾는다.
Health Check Worker는 lastSeenAt을 기준으로 위치 보고가 끊긴 기사를 감지한다.
Ride Service는 Kafka 이벤트를 구독해 호출 상태를 Ride DB에 반영한다.
Notification Service는 Kafka 이벤트를 구독해 승객과 기사에게 실시간 알림을 전달한다.
```

저장소는 데이터 성격에 따라 분리했다.

```
실시간 기사 위치 검색 → Redis GEO
기사 상태 및 lastSeenAt → Redis Hash
호출 상태 정합성 → PostgreSQL
이벤트 전달 → Kafka
```


이 구조를 통해 위치 업데이트가 많은 환경에서도 주변 기사를 빠르게 검색할 수 있고, 오래된 위치로 인한 잘못된 매칭을 줄일 수 있다. 또한 Kafka 기반 이벤트 흐름을 통해 위치 변경, 기사 오프라인 감지, 매칭 결과, 호출 상태 변경을 비동기적으로 처리할 수 있다.

## 개인 후기

이번 택시 호출 서비스 설계를 진행하며 가장 크게 배운 점은, 단순히 기능을 나누는 것이 아니라 **각 작업의 성격에 맞는 서비스 책임과 데이터 저장 구조를 선택해야 한다는 것**이었습니다.

특히 실시간 위치 추적이 포함된 서비스에서는 성능, 비용, 구조의 복잡도, 운영 복잡도를 함께 고려해야 한다는 점을 처음으로 깊게 고민해볼 수 있었습니다. 예를 들어 기사 위치처럼 최신성이 중요하고 업데이트가 잦은 데이터는 RDBMS보다 Redis GEO와 Redis Hash 같은 자료구조가 더 적합할 수 있다는 점을 배웠습니다.

아직 Redis GEO, Kafka, 이벤트 기반 구조를 완전히 흡수한 상태에서 진행한 설계는 아니었고, 시스템 전체를 바라보는 관점도 부족하다고 느꼈습니다. 그래도 이번 과제를 통해 실시간 위치 기반 시스템에서 어떤 데이터는 빠르게 처리하고, 어떤 데이터는 정합성을 중심으로 관리해야 하는지 고민해볼 수 있었습니다.

다음 설계에서는 각 기술을 왜 선택했는지 더 명확하게 설명할 수 있을 만큼 학습한 뒤, 더 확신 있는 아키텍처를 제안해보고 싶습니다.

## 추가 학습한 개념
이번 설계에서 공부한 주요 기술 개념입니다.

### Redis GEO
Redis GEO는 위도/경도 기반 위치 데이터를 저장하고, 특정 좌표 기준 반경 내 객체를 검색할 수 있는 Redis 자료구조다.

택시 호출 서비스에서는 다음 용도로 사용된다.

```text
- 가용 기사 위치 저장
- 승객 픽업 위치 주변 기사 검색
- 직선거리 기준 후보 기사 조회
```

대표 명령어
```text
GEOADD
GEOSEARCH
GEOREM
```

### Redis Hash
Redis Hash는 하나의 key 아래 여러 field-value 쌍을 저장할 수 있는 자료구조다.

본 설계에서는 기사 상태 메타데이터 저장에 사용한다.

```text
driver:status:{driverId}
- status
- lastSeenAt
- speed
- heading
- currentRideId
```

대표 명령어
```text
HSET
HGET
HGETALL
```

### Kafka
Kafka는 이벤트 스트림을 처리하기 위한 메시지 큐다.
서비스 간 직접 호출을 줄이고, 상태 변경이나 위치 변경 같은 이벤트를 비동기적으로 전달하는 데 사용한다.

본 설계에서는 다음 이벤트를 Kafka로 전달한다.
```text
- 기사 위치 변경 이벤트
- 위치 보고 끊김 감지 이벤트
- 매칭 결과 이벤트
- 호출 상태 변경 이벤트
```
Kafka를 사용하면 서비스 간 결합도를 낮출 수 있지만, 이벤트 중복 처리와 장애 복구에 대한 고려가 필요하다.

### Health Check Worker
Health Check Worker는 주기적으로 lastSeenAt을 확인해 위치 보고가 끊긴 기사를 감지하는 백그라운드 작업이다.

역할은 다음과 같다.
```text
- Redis Hash에서 lastSeenAt 확인
- 오래된 기사 Redis GEO에서 제거
- 기사 오프라인 감지 이벤트 발행
- Ride Service가 후속 상태 처리할 수 있도록 이벤트 전달
```

### Event-driven Architecture
이벤트 기반 아키텍처는 서비스 간 직접 호출 대신 이벤트를 발행하고 구독하는 구조다.

본 설계에서는 다음과 같은 장점이 있다.
```text
- 위치 저장과 알림 전송 분리
- 매칭 결과 처리와 호출 상태 변경 분리
- 기사 오프라인 감지와 후속 처리 분리
- 피크 시간대 이벤트 처리 완충
```
다만 이벤트 순서, 중복 처리, 실패 재처리 전략이 필요하므로 운영 복잡도도 함께 증가한다.

## 최종 요약
실시간 택시 호출 서비스에서는 기사 위치를 Redis GEO에 저장해 주변 가용 기사를 빠르게 검색하고, Redis Hash의 상태 정보와 lastSeenAt을 함께 활용해 오래된 위치나 이용 불가능한 기사를 제외한다. 이후 Kafka 기반 이벤트 흐름을 통해 위치 변경, 기사 오프라인 감지, 매칭 결과, 호출 상태 변경을 비동기적으로 처리함으로써 실시간성과 상태 일관성을 함께 확보한다.
