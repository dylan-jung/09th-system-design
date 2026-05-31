# Week 2 과제: 택시 호출 서비스 설계
## 1. 문제 이해 및 설계 범위 확정

### 시나리오

승객이 호출하면 주변 빈 택시를 찾아 매칭하고, 매칭된 택시가 픽업 지점에 도착할 때까지 승객은 택시 위치와 도착 예상 시간을 실시간으로 확인한다. 픽업 이후 목적지에 도착할 때까지 위치 추적이 이어진다.

### 설계 범위 (In / Out of Scope)

| 포함 (In Scope) | 제외 (Out of Scope) |
|---|---|
| 호출 시점부터 도착 완료 시점까지의 실시간 흐름 | 회원가입 · 인증 · 기사 등록 절차 |
| 기사·승객 위치 추적 | 이용 이력 |
| 매칭 로직 | 운행 종료 후 요금 산정 · 결제 · 리뷰 · 정산 |
| 이상 상황 처리 (앱 종료, 네트워크 단절, 취소 등) | 도로망 데이터 / 경로 탐색 알고리즘 |

### 시스템 구성 전제
- 외부 의존: 지도 시스템(OOO맵, 경로·ETA), 회원 시스템, 결제 시스템
- 기사·승객 모두 로그인 상태, 결제 수단 등록 완료 상태로 가정
- 외부 시스템 의존에서 오는 문제는 본 서비스가 책임진다

### 기능 요구사항
- 운행중인 기사의 위치는 4초마다 업데이트
- 승객이 호출 시 지정 반경 내 빈 택시를 ETA 기반으로 매칭
- 매칭 후 픽업 이동 중: 승객 화면에 택시 위치 + 픽업 지점까지 경로·ETA 실시간 표시
- 운행 중: 승객 화면에 현재 위치 + 목적지까지 경로·ETA 실시간 표시
- 이상 상황(앱 종료·네트워크 단절·호출 취소)에서 시스템이 일관된 상태 유지

### 비기능 요구사항 (시간 / 지연 목표)

| 항목 | 목표 |
|---|---|
| 호출 접수 응답 시간 | 호출 요청 → "기사 검색 중" 진입까지 **2초 이내** |
| 매칭 완료 시간 | 평균 **30초 이내**, 5분 초과 시 실패 처리 |
| 위치 추적 갱신 지연 | 기사 위치 변화 → 승객 화면 반영 평균 **5초 이내** |

### 개략적 규모 추정

| 항목 | 수치 |
|---|---|
| 서비스 지역 | 단일 대도시권 |
| 누적 가입 승객 | 약 2,000,000명 |
| MAU / DAU | 약 800,000명 / 약 200,000명 |
| 누적 가입 기사 | 약 50,000명 |
| 동시 운행 기사 | 10,000명 |
| 일일 호출 수 | 약 200,000건 |
| 피크 시간 호출 집중도 | 평균 대비 **5배 이상** |
| 피크 시간대 | 평일 출근 07:30–09:30 / 퇴근 18:00–20:00 / 금·토 심야 23:00–02:00 |


### 본인이 추가로 둔 가정

* 운행중인 기사의 위치는 4초마다 업데이트 된다.

기사 위치 업데이트 규모

- 초당 업데이트
10000명 / 4초 = 2500 Writes/sec
피크 시간 -> 12500 Writes/sec

- 한 명의 기사 위치 업데이트에 대한 데이터 페이로드
기사ID(8B) + 위도(8B) + 경도(8B) + timestamp(8B) + 메타데이터(8B) = 약 40B 예상

- 전체 기사 위치 업데이트에 대한 데이터 페이로드
10000명 * 40B = 약 400 KB 예상
피크 시간 -> 400KB * 5 = 약 2MB 예상

---

## 2. 개략적 설계안 제시 및 동의 구하기

### 개략적 아키텍처 다이어그램

<img width="1771" height="878" alt="week2 drawio" src="https://github.com/user-attachments/assets/c613faff-1c2c-48c1-a824-b81e12ccea22" />

### 핵심 흐름

**운행중인 기사의 위치 업데이트**

기사의 app은 api 서버에 4초마다 현재 위치를 전송한다.

api 서버는 받은 위치 데이터를 Kafka에 publish한다.

Location Service는 Kafka에서 기사 위치 데이터를 가져와서 GPS 좌표를 H3 cell ID(hid)로 변환한 후, Redis에 driver 위치를 업데이트한다.

**승객 app이 탑승 요청을 한 경우**

탑승 요청은 로드밸런서를 거쳐 api 서버로 들어오고, api 서버가 이를 매칭 서비스로 전달한다.

매칭 서비스는 승객의 위치로 H3 cell ID(hid)를 계산한다.

해당 cell과 인접 cell들(K-ring)을 Redis에서 조회하고, available 상태인 driver 후보들을 가져온다.

각 후보에 대해 ETA를 계산하고 최적의 driver를 선정한다.

선정된 driver에게 api 서버(Websocket)를 통해 탑승 요청 알림을 push한다.

  - 거부 시: 다음 후보 driver를 선정해 알림 push (반복)
  - 승낙 시:
    1. api 서버(Websocket)를 통해 rider에게 매칭 결과(driver 정보 + 픽업 ETA)를 즉시 push
    2. Trip Service가 매칭 결과를 받아 Trip DB에 기록

**매칭 후 driver 위치를 rider에게 실시간 전달**

forward service가 Kafka에서 기사 위치 데이터를 가져온다.

매칭이 성사된 driver-rider 쌍을 파악하고 rider에게 매칭된 driver의 위치정보를 WebSocket으로 push한다.

## 3. 상세 설계

### 3-1. 빠른 검색을 위한 공간 인덱싱
승객이 탑승 요청을 하는 경우 승객과 인접한 위치의 기사들을 검색해야한다.
이 때 공간 인덱싱을 통해 검색을 효율적으로 할 수 있다.
이번 과제에서는 uber에서 만든 geohash 방식의 H3 라이브러리를 활용한다.
location service에서는 H3를 사용하여 (lat, long)을 hid로 변환한다.

<img width="432" height="386" alt="image" src="https://github.com/user-attachments/assets/481eb84f-892c-43ff-a838-44c2aa7b6e80" />


- 셀 내부에 위치한 기사 목록을 O(1)에 검색할 수 있다.
- 주어진 셀에 대해 인접한 셀을 찾는 연산도 O(1)에 할 수 있다. 따라서 인접한 기사를 찾지 못한 경우에 인접한 셀로 범위를 확장해가며 추가 검색을 효율적으로 할 수 있다.

### 3-2. 기사 위치 업데이트 주기

- 상태별(대기 중 · 매칭 중 · 픽업 이동 중 · 운행 중)로 어떻게 다르게 둘 것인가?

driver 상태:
- offline:      앱 끔, 운행 안 함
- available:    온라인, 운행 가능 (매칭 대상 ✅)
- en_route:     배차 받고 픽업 장소로 가는 중 (매칭 대상 ❌)
- on_trip:      승객 태우고 운행 중 (매칭 대상 ❌)

|상태|위치 업데이트 주기|이유|
|---|---|---|
|offline|보내지 않음|매칭 대상 아님|
|available|4초|매칭 정확도용|
|en_route|3초|rider가 픽업 위치 추적|
on_trip|5초|rider는 이미 차에 있음, 정밀도 덜 중요|


### 3-3. 기사 위치 저장 — 어디에, 어떤 형태로?

Kafka (driver_locations topic)
- 4초마다 업데이트 되는 기사 위치가 먼저 저장되는 곳
- 좌표 형식으로 저장
- 여러 consumer가 fan-out (Location Service, Forward service)
- 영속성: 보관 기간 짧음 (수 시간), 단기 buffer + fan-out 용도

Redis (driver hot store)
- 매칭시 인접 기사를 빠르게 찾는 용도
- location service를 통해 인덱싱된 데이터가 저장됨
- key: H3 cell_id, value: driver 인덱스 set
- 데이터 구조 예시
    - cell:8930e1d8e2bffff:available → SET { D111, D123, ..., D789 }
    - cell:8930e1d8e2bffff:en_route  → SET { D456 }

### 3-4. 검색 반경 확장 방식

인접 기사 검색 결과가 0건일 때의 반경 확장 정책으로 k-ring 방식을 활용한다.

<img width="642" height="478" alt="image-1" src="https://github.com/user-attachments/assets/9d1920ba-7198-4d17-97f7-7dde7a7bb6d4" />

```
1차 검색: K=1 (자신 + 인접 6개 cell)
  └─ 후보 있음 → 매칭 진행
  └─ 후보 0건 → 2차 검색

2차 검색: K=2 (반경 확장)
  └─ 후보 있음 → 매칭 진행
  └─ 후보 0건 → 3차 검색 또는 대기열로

3차 검색 또는 대기열:
  - K=3 까지 확장 (반경 ~수 km)
  - 그래도 없으면: "근처에 차량이 없습니다" + 대기열 등록
  - 일정 시간 내 신규 driver available 시 자동 재시도
```

### 3-5. 수요 폭주 예측 시 대응

Pre-warming

- 인근 H3 cell에 위치한 driver에게 미리 알림 push ("잠실역 인근, 1시간 뒤 요청 폭주 예상")
- Matching service, api 서버 인스턴스 수 사전 증설

Backpressure / Queueing

- Matching service가 overload 시 ride request를 Kafka에 buffer
"예상 대기 시간 X분"을 rider에게 표시 → 자발적 분산


H3 cell split

- 레디스 노드들이 H3 cell id를 key로 샤딩 되어있다고 가정

- H3는 해상도(resolution) 0~15 까지 있다. 숫자가 클수록 셀이 작아짐. 우버는 보통 res 8이나 9를 쓴다고 알려져 있다.
요청이 몰리는 지역만 cell의 Resolution을 동적으로 올리면 쪼개진 cell 내의 부하가 여러 redis 노드에 분산 될 것이다.

|Resolution|셀 평균 면적|
|---|---|
|0|4,250,000 km² (대륙 크기)|
|7|5.16 km² (동네)|
|9|0.1 km² (블록)|
|15|0.9 m² (사람 한 명)|


Split 전:
```
단일 res 8 cell "882a1072b3fffff" 가 잠실역 일대 전부 포함
→ key "drivers:cell:882a1072b3fffff"
→ CRC16 해시 → slot 4521 → Node-3
→ 모든 잠실역 일대 매칭 요청이 Node-3에 집중. Node-3만 CPU 95%, 나머지 노드는 한가함.
```
Split 후 (res 8 → res 9, 자식 7개):
```
잠실역 일대 = 7개 res 9 cell
  892a1072b03ffff → slot 1247  → Node-1
  892a1072b07ffff → slot 8932  → Node-6
  892a1072b0bffff → slot 15003 → Node-10
  892a1072b0fffff → slot 442   → Node-1
  892a1072b13ffff → slot 6781  → Node-5
  892a1072b17ffff → slot 11290 → Node-8
  892a1072b1bffff → slot 3104  → Node-2
→ 같은 양의 트래픽이 7개 노드로 분산.
```

---

### 개인적 의견 / 사례 공유 / 추가 학습

trade-off를 생각하지 못하고, 기능만 구현하다가 끝났다.
다음 주차부터는 시간을 더 많이 투자해서 여러가지 선택지에 대한 trade-off 비교해가며 아키텍처를 짜는 것이 목표.

---

## 📚 참고 자료

- 가상 면접 사례로 배우는 대규모 시스템 설계 기초2 2장
- https://github.com/uber/h3
- https://www.youtube.com/watch?v=Tq72Pq0entg
