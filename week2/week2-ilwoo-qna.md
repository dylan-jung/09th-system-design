# Week 2 발표 준비 Q&A

## 1. MSA로 나누면 부하가 큰 부분만 스케일 아웃하면 되는가?

큰 방향은 맞다. 다만 MSA로 나누었다고 자동으로 확장성이 생기는 것은 아니고, 각 컴포넌트의 부하 특성에 맞게 확장 전략을 다르게 가져가야 한다.

- API Gateway, Trip Service, Location Service는 stateless하게 만들면 서버 인스턴스를 늘려 수평 확장할 수 있다.
- Preference Dispatch Service는 Kafka consumer group으로 worker 수를 늘려 병렬 처리할 수 있다.
- Realtime Gateway는 WebSocket/MQTT 연결을 오래 유지하므로 연결 수 기준으로 확장해야 한다.
- Redis Geo, PostgreSQL 같은 저장소는 단순히 서버만 늘리는 것으로 끝나지 않고 key 분산, 파티셔닝, replica, managed DB 같은 전략이 필요하다.

발표 답변:

> 부하가 많은 서비스는 stateless하게 설계해 수평 확장하고, 위치 검색이나 Trip 상태 저장처럼 상태를 가진 저장소는 Redis Cluster, 지역별 key 분리, PostgreSQL 관리형 서비스와 파티셔닝 전략을 함께 고려합니다.

## 2. API Gateway와 Realtime Gateway도 서비스인가?

둘 다 배포 가능한 서비스로 보면 된다. Spring Boot로 구현할 수도 있다.

- API Gateway는 HTTPS 기반 REST 요청의 입구다. 호출 생성, 선호도 저장 요청 같은 짧은 HTTP 요청을 받아 내부 서비스로 전달한다.
- Realtime Gateway는 WebSocket/MQTT 연결을 유지하며 서버가 앱으로 콜 제안, 매칭 결과, 위치 상태를 push하는 역할이다.

발표 답변:

> 두 Gateway 모두 서비스입니다. API Gateway는 일반 REST 요청을 처리하는 입구이고, Realtime Gateway는 오래 유지되는 실시간 연결을 관리하는 입구입니다. MVP에서는 Spring Boot로 구현할 수 있지만, 연결 수가 커지면 Realtime Gateway는 Netty, Node.js, Go, MQTT Broker 같은 이벤트 기반 구조를 고려할 수 있습니다.

## 3. PostGIS와 Redis Geo는 같은 역할인가?

둘 다 위치 데이터를 다루지만 역할이 다르다.

| 구분 | Redis Geo | PostGIS |
| --- | --- | --- |
| 기반 | Redis | PostgreSQL 확장 |
| 주 용도 | 최신 위치 기반 빠른 반경 검색 | 복잡한 공간 데이터 저장/분석 |
| 데이터 성격 | 자주 바뀌는 hot 데이터 | 영속 저장이 필요한 공간 데이터 |
| 예시 | 근처 기사 50명 조회 | 목적지가 선호지역 폴리곤 안인지 판단 |

우리 설계에서는 다음처럼 나눈다.

- Redis Geo: 기사들의 최신 위치
- PostGIS: 기사들의 선호지역, 기피지역, 복귀지역 같은 공간 설정

발표 답변:

> Redis Geo는 실시간 매칭을 위한 최신 기사 위치 검색에 사용하고, PostGIS는 기사 선호지역처럼 영속성과 복잡한 공간 질의가 필요한 데이터에 사용합니다.

## 4. H3는 무엇이고 Redis Geo와 어떻게 다른가?

H3는 Uber가 만든 공간 인덱싱 라이브러리다. 위도/경도를 육각형 셀 ID로 변환해서 지도 공간을 cell 단위로 다룰 수 있게 해준다.

Redis Geo는 "현재 위치 기준 반경 3km 안의 기사"를 바로 찾는 방식에 가깝다.

H3는 "이 위치가 속한 셀과 주변 셀에 있는 기사"를 찾는 방식에 가깝다.

| 역할 | 현재 설계 | H3 기반 설계 |
| --- | --- | --- |
| 근처 기사 찾기 | Redis Geo 반경 검색 | 현재 위치를 H3 cell로 변환 후 주변 cell 조회 |
| 선호지역 판단 | PostGIS 폴리곤 질의 | 선호지역을 H3 cell 목록으로 변환 후 비교 |
| 핫스팟 대응 | Redis Geo + 반경 확장 | cell 단위 수요/공급 집계 |
| 확장 방식 | Redis key/cluster 중심 | H3 cell 기준 파티셔닝 가능 |

발표 답변:

> MVP에서는 Redis Geo가 단순하고 충분합니다. 다만 콘서트장, 역세권처럼 특정 지역 핫스팟이 반복되거나 지역별 수요/공급 집계가 중요해지면 H3 cell 기반으로 공간을 나누는 전략을 도입할 수 있습니다.

## 5. H3 cell ID는 보통 어디에 저장하는가?

H3 자체는 DB가 아니라 위치를 cell ID로 바꾸는 인덱싱 방식이다. 저장 위치는 데이터 성격에 따라 다르다.

| 데이터 | 저장 위치 | 이유 |
| --- | --- | --- |
| 기사 선호지역 H3 cell 목록 | PostgreSQL | 잘 바뀌지 않고 영속 저장 필요 |
| 기사 현재 위치 H3 cell | Redis | 자주 바뀌는 hot 데이터 |
| 지역별 수요/공급 집계 | Redis, Kafka, 분석 DB | 실시간 집계와 분석 목적 |
| 과거 위치 이력 | TSDB, Data Warehouse | 분석, 예측, 모델 학습 목적 |

발표 답변:

> H3를 도입한다면 선호지역처럼 자주 바뀌지 않는 cell 목록은 RDB에 저장하고, 기사 현재 위치처럼 자주 바뀌는 cell 정보는 Redis에 저장하는 방식이 자연스럽습니다.

## 6. 동시 수락은 어떻게 처리하는가?

Top-N 기사에게 동시에 콜을 제안하면 여러 기사가 거의 동시에 수락할 수 있다. 이때 한 콜에 여러 기사가 매칭되면 안 된다.

이를 막기 위해 Trip Service는 수락 시 다음과 같은 조건부 UPDATE를 수행한다.

```sql
UPDATE trip
SET
  driver_id = :driver_id,
  state = 'MATCHED',
  matched_at = NOW()
WHERE trip_id = :trip_id
  AND state = 'MATCHING';
```

핵심은 `state = 'MATCHING'`인 경우에만 업데이트한다는 점이다.

- `affected_rows = 1`: 실제로 row가 변경됨. 내가 첫 수락자이므로 매칭 성공.
- `affected_rows = 0`: 변경된 row가 없음. 이미 다른 기사가 먼저 수락했으므로 실패.

PostgreSQL은 같은 trip row에 대한 UPDATE를 row lock으로 직렬화한다. 따라서 거의 동시에 요청이 와도 먼저 처리된 하나만 성공한다.

발표 답변:

> 별도 Redis lock을 두지 않고 PostgreSQL 조건부 UPDATE로 first-accept-wins를 처리합니다. DB가 같은 row에 대한 UPDATE를 직렬화하므로, 먼저 성공한 기사만 affected_rows가 1이 되고 나머지는 0이 됩니다.

## 7. 조건부 UPDATE가 좋은 이유는 무엇인가?

조건부 UPDATE는 "상태 확인"과 "상태 변경"을 쿼리 한 번으로 처리한다.

나쁜 방식은 먼저 SELECT로 상태를 확인하고 나중에 UPDATE하는 것이다. SELECT와 UPDATE 사이에 다른 기사가 먼저 수락할 수 있기 때문이다.

좋은 방식은 다음처럼 DB에게 한 번에 요청하는 것이다.

```sql
UPDATE trip
SET state = 'MATCHED', driver_id = :driver_id
WHERE trip_id = :trip_id
  AND state = 'MATCHING';
```

발표 답변:

> 조건부 UPDATE는 상태 확인과 변경을 하나의 원자적 연산처럼 처리할 수 있어 동시성 문제가 줄어듭니다. 우리 규모에서는 Redis SETNX 같은 별도 lock 저장소 없이 PostgreSQL row lock만으로 충분하다고 판단했습니다.

## 8. 외부 지도 API의 ETA batch 호출이란 무엇인가?

Batch 호출은 여러 요청을 하나씩 보내지 않고 한 번에 묶어서 보내는 방식이다. Bulk 호출과 비슷하게 이해하면 된다.

예를 들어 후보 기사 10명의 픽업 ETA를 계산해야 할 때 다음 두 방식이 있다.

- 개별 호출: 기사 10명에 대해 API 10번 호출
- batch 호출: 기사 10명의 위치를 묶어서 API 1번 호출

발표 답변:

> 후보 기사마다 ETA API를 개별 호출하면 외부 API 호출 수와 지연이 커지기 때문에, 후보 기사들의 위치를 묶어 batch 또는 bulk 형태로 ETA를 계산합니다.

## 9. MQTT는 WebSocket보다 가벼운 프로토콜인가?

대체로 그렇다. 더 정확히는 MQTT는 작은 메시지를 자주 주고받는 모바일/IoT 환경에 특화된 가벼운 pub/sub 메시징 프로토콜이고, WebSocket은 범용 양방향 통신 채널이다.

| 구분 | WebSocket | MQTT |
| --- | --- | --- |
| 성격 | 양방향 연결 통신 | pub/sub 메시징 프로토콜 |
| 메시지 구조 | 직접 정의 필요 | topic, QoS 제공 |
| 모바일 위치 업데이트 | 가능 | 매우 적합 |
| 운영 난이도 | 상대적으로 단순 | Broker 운영 필요 |

발표 답변:

> 기사 위치 업데이트처럼 작고 잦은 메시지에는 MQTT가 더 적합할 수 있습니다. 다만 MVP에서는 WebSocket만으로도 구현 가능하므로 운영 단순성을 위해 WebSocket으로 시작하고, 위치 스트림 부하가 커지면 MQTT Broker를 도입할 수 있습니다.

## 10. MQTT도 WebSocket 기반인가?

꼭 그렇지는 않다. MQTT는 원래 TCP 위에서 동작하는 별도 프로토콜이다.

- MQTT over TCP: 일반적인 MQTT 기본 방식
- MQTT over WebSocket: 브라우저나 방화벽 환경에서 사용하기 위해 WebSocket 위에 MQTT를 얹은 방식
- WebSocket: MQTT와 별개의 범용 양방향 통신 프로토콜

발표 답변:

> MQTT는 기본적으로 TCP 기반 프로토콜입니다. 다만 브라우저 환경에서는 MQTT over WebSocket 형태로 사용할 수도 있습니다.

## 11. 기사 위치 업데이트 주기는 항상 같아야 하는가?

아니다. 기사 상태에 따라 위치 업데이트 주기를 다르게 가져가면 배터리, 네트워크, 서버 부하를 줄일 수 있다.

| 기사 상태 | 위치 업데이트 주기 예시 |
| --- | --- |
| 대기 중 | 10~15초 |
| 콜 제안 중 | 3~5초 |
| 픽업 이동 중 | 2~3초 |
| 승객 탑승 후 운행 중 | 2~5초 |
| 정지 상태 | backoff 적용 |

발표 답변:

> 모든 기사에게 동일한 위치 업데이트 주기를 강제하지 않고, 상태에 따라 주기를 조절합니다. 대기 중에는 느리게, 매칭이나 픽업 이동 중에는 빠르게 보내도록 해서 실시간성과 비용 사이의 균형을 맞춥니다.

## 12. TSDB는 무엇인가?

TSDB는 Time Series Database, 즉 시계열 데이터베이스다. 시간 순서대로 계속 쌓이는 데이터를 저장하고 조회하는 데 특화되어 있다.

예시:

- 기사 위치 이력
- 지역별 기사 밀도 변화
- 호출 수 변화
- 매칭 성공률 변화
- ETA 예측 성능 모니터링

대표적인 TSDB로는 InfluxDB, TimescaleDB, Prometheus, VictoriaMetrics 등이 있다.

우리 MVP에서는 실시간 매칭에 최신 위치만 필요하므로 Redis Geo를 사용한다. 위치 이력 분석이나 수요 예측이 필요해지면 Kafka의 위치 이벤트를 TSDB나 Data Warehouse로 적재할 수 있다.

발표 답변:

> TSDB는 실시간 매칭용이라기보다 위치 이력, 호출량 변화, 수요 예측 같은 분석용 저장소입니다. MVP에서는 최신 위치만 필요하므로 Redis Geo만 사용하고, 분석 요구가 생기면 TSDB를 추가할 수 있습니다.

## 13. Redis Cache는 정확히 무엇인가?

Redis Cache는 원본 DB에 있는 데이터를 더 빠르게 읽기 위해 Redis에 임시 복사해두는 것이다.

우리 설계에서 기사 선호도 원본은 PostgreSQL/PostGIS에 있다. 매칭 시 후보 기사 50명의 선호도를 매번 DB에서 읽으면 부하가 커질 수 있으므로 Redis Cache에 자주 읽는 선호도 데이터를 저장할 수 있다.

Cache-aside 흐름:

1. 캐시에 있으면 Redis에서 바로 읽는다.
2. 캐시에 없으면 PostgreSQL에서 읽는다.
3. 읽은 값을 Redis에 저장한다.
4. 다음 요청부터 Redis에서 읽는다.

발표 답변:

> Redis Cache는 원본 DB가 아니라 빠른 임시 복사본입니다. MVP에서는 DB 직접 조회로 시작하고, 선호도 조회가 병목이 되면 cache-aside 방식으로 Redis Cache를 추가할 수 있습니다.

## 14. Redis Cache도 스케일 아웃하는가?

가능하다. 일반적으로 Redis Cluster나 샤딩을 사용한다.

Redis Cluster는 key를 여러 노드에 나눠 저장한다. 예를 들어 다음 key들이 여러 master node에 분산될 수 있다.

```text
driver_preference:driver_123
driver_preference:driver_456
driver_preference:driver_789
```

다만 캐시는 원본 데이터가 아니므로 장애가 나도 DB에서 다시 채울 수 있다. Trip DB처럼 강한 정합성이 필요한 저장소보다는 운영 부담이 작다.

발표 답변:

> Redis Cache도 Redis Cluster를 통해 key 기반으로 분산할 수 있습니다. 캐시는 원본이 아니므로 장애 시 DB에서 다시 채울 수 있고, 처리량과 메모리 용량이 부족해질 때 cluster 구성을 고려합니다.

## 15. DB는 보통 관리형 서비스를 쓰는가?

실무에서는 PostgreSQL 같은 DB를 직접 운영하기보다 AWS RDS/Aurora, Google Cloud SQL, Azure Database for PostgreSQL 같은 관리형 서비스를 많이 사용한다.

이유:

- 백업
- 장애 복구
- 모니터링
- 보안 패치
- replica 구성
- failover
- 스토리지 확장

발표 답변:

> PostgreSQL/PostGIS는 직접 운영하기보다 관리형 DB를 사용하는 것이 현실적입니다. 관리형 서비스를 통해 백업, 장애 조치, replica, 모니터링을 위임하고, 애플리케이션은 도메인 로직에 집중할 수 있습니다.

## 16. Redis Cluster는 내부적으로 샤딩처럼 동작하는가?

맞다. Redis Cluster는 key 공간을 16,384개의 hash slot으로 나누고, 각 slot을 여러 Redis master node에 분산 배치한다.

예시:

```text
Redis Master A: slot 0 ~ 5460
Redis Master B: slot 5461 ~ 10922
Redis Master C: slot 10923 ~ 16383
```

각 key는 hash slot에 배정되고, 그 slot을 담당하는 master node에 저장된다.

```text
driver_preference:123 -> slot 3521 -> Master A
driver_preference:456 -> slot 10982 -> Master B
```

주의할 점은 Redis Geo다. 모든 기사 위치를 하나의 key에 넣으면 Redis Cluster를 써도 그 key는 하나의 slot에만 저장된다.

```text
active_drivers
```

따라서 대규모가 되면 지역이나 H3 cell 기준으로 key를 나누는 것이 좋다.

```text
active_drivers:gangnam
active_drivers:jamsil
active_drivers:h3:8830e1d8
```

발표 답변:

> Redis Cluster는 hash slot 기반의 key-level 자동 샤딩 구조입니다. 일반 캐시 key는 자연스럽게 분산되지만, Redis Geo를 하나의 active_drivers key로만 쓰면 한 노드에 몰릴 수 있습니다. 그래서 규모가 커지면 지역이나 H3 cell 기준으로 key를 나눕니다.

