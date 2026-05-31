# 커머스 추천 지면 시스템 설계 (패션 도메인)

## 1. 문제 이해 및 설계 범위 확정

### 시나리오

패션 종합 이커머스 (무신사 / 29CM 류) 의 메인 페이지 "당신을 위한 추천" 지면. 2 × 5 그리드로 10개 상품 노출. 핵심 지표는 **CTA (장바구니 담기) 전환율**.

**페르소나**: 20-30대 패션 관심 사용자, 매일 또는 주 3-4회 방문 (재방문율 높음)

**도메인 특성** (시스템 결정에 영향):
- 매일 신상 입고 → 콘텐츠 신선도 중요
- 매일 들어옴 → "어제 본 상품 dedup" 필요
- 사용자 취향 강함 (스타일, 브랜드)

### 전제

- 추천 모델은 별도 ML 팀이 관리. 모델 자체와 학습 파이프라인은 out of scope
- 상품/재고/가격 정보는 별도 상품 시스템에서 조회 가능
- 추천 시스템은 **이벤트 수집, 후보 조회, 랭킹, 응답 제공, 성과 측정** 책임

### 규모

| 항목 | 수치 |
|------|------|
| MAU / DAU | 10M / 500K |
| 활성 사용자 (어제+오늘 방문) | ~600K |
| 일일 추천 API 요청 | ~1M |
| 일일 이벤트 | ~15M |
| 평균 추천 QPS | 12 |
| 피크 추천 QPS | 150~300 |
| 피크 이벤트 수집 QPS | 1,000~3,000 |

### 비기능 요구사항

| 항목 | 목표 |
|------|------|
| 추천 API 응답 | p95 200ms 이하 |
| 이벤트 수집 응답 | p95 100ms 이하 |
| 가용성 | 월 99.9% |
| 이벤트 유실 | 0.1% 이하 |
| 피크 트래픽 | 평시 대비 3배 |
| 장애 대응 | fallback 응답 보장 |

---

## 2. 개략적 설계안

### 핵심 관찰

이 시스템에는 **완전히 다른 두 개의 트래픽 흐름**이 있다.

| 흐름 | 특성 | SLA |
|------|------|-----|
| **추천 요청** (읽기) | 저지연, 캐시 친화 | p95 200ms |
| **이벤트 수집** (쓰기) | 대용량, 유실 방지 | p95 100ms |

→ **두 파이프라인을 독립 서비스로 분리**. 한 서버에 묶으면 쓰기 부하가 읽기 응답에 영향.

### 핵심 흐름

**추천 요청**:
```
사용자 진입 → 추천 API → Redis 캐시 조회
  → hit: 즉시 반환
  → miss: 후보 생성 + 랭킹 → 캐시 저장 → 반환
  → Redis 장애: in-memory fallback (인기 상품)
```

**이벤트 수집**:
```
사용자 행동 → 수집 API → Kafka (acks=all)
  → Hot Path: 실시간 컨슈머 → Redis L3 갱신 (멱등)
  → Cold Path: 배치 컨슈머 → ClickHouse (자동 dedup)
                                ↓
                          ML 팀이 학습 + 6시간마다 후보 push
```

**CTA Funnel**:
```
추천 API 가 impression_id 발급
  → 노출/클릭/장바구니 이벤트가 같은 impression_id 포함
  → ClickHouse 에서 join 으로 funnel 분석
```

### 아키텍처

```
        ┌─── 사용자 ───┐
        │              │
   추천  │              │ 이벤트
        ▼              ▼
   [추천 API]    [이벤트 수집 API]
        │              │ acks=all
        ▼              ▼
   [Redis 캐시]      [Kafka]
   L1/L2/L3            │
        │       ┌──────┴──────┐
        │       ▼             ▼
        │  [실시간 컨슈머] [배치 컨슈머]
        │  (Hot Path)    (Cold Path)
        │       │             │
        │       ▼             ▼
        │  [Redis L3]   [ClickHouse]
        │  (멱등 갱신)  (raw, 자동 dedup)
        │                     │
        │              [ML 팀: 학습]
        │                     │
        │              [Redis L2 push]
        ▼   (6시간 배치)      
   [In-memory fallback]
   (Redis 장애 시,
    외부 의존 0)
```

### 컴포넌트 책임

| 컴포넌트 | 책임 |
|----------|------|
| 추천 API | 캐시 조회 → 후보 생성 → 랭킹 → 응답 |
| 이벤트 수집 API | 검증 → Kafka 적재 → 즉시 응답 |
| Redis | L1 (응답 캐시), L2 (후보 풀), L3 (사용자 프로필) |
| Kafka | 이벤트 버퍼 (at-least-once), 다중 컨슈머 분산 |
| 실시간 컨슈머 | 멱등 자료구조로 Redis L3 갱신 |
| 배치 컨슈머 | ClickHouse 적재 |
| ClickHouse | raw 이벤트 (ReplacingMergeTree 자동 dedup), ML 학습 소스 |
| In-memory fallback | Redis 장애 시 응답. 외부 의존 0 |

---

## 3. 상세 설계

핵심 토픽 5개:

### 3-1. 이벤트 수집

#### 이벤트 종류

| 종류 | 빈도 | 비고 |
|------|------|------|
| impression | 가장 많음 | 노출 시 |
| click | 중간 (CTR 3-8%) | 상품 클릭 |
| add_to_cart | 적음 (0.5-2%) | **핵심 지표** |

#### 스키마

```json
{
  "event_id": "uuid",        // dedup 키
  "event_type": "click",
  "user_id": "uid",          // 로그인 시
  "device_id": "did",        // 항상
  "timestamp": 1700000000,
  "impression_id": "imp_x",  // funnel 연결
  "product_id": "prod_y",
  "position": 3
}
```

#### 보장 수준: At-least-once + 저장소 dedup

**핵심 결정**: exactly-once 안 추구. **at-least-once + 저장소가 dedup 책임**.

**유실 방지**:
- 클라이언트: localStorage 버퍼링 + `sendBeacon` (페이지 이탈)
- 수집 API → Kafka: `acks=all` (모든 ISR 받은 후 ack)
- 컨슈머: 처리 성공 후 offset commit

**중복 처리**: 컨슈머는 그냥 INSERT/UPDATE. 저장소가 dedup:
- **ClickHouse**: `ReplacingMergeTree(ts) ORDER BY event_id` → background merge 시 자동 dedup
- **Redis L3**: `ZADD` 같은 멱등 자료구조 (같은 멤버는 score 만 갱신)

→ 별도 dedup 인프라 (Redis SET NX 등) 불필요.

#### 클라이언트 배치 전송

- impression (많음): 5초 또는 10개 단위 배치
- click / add_to_cart (중요): 즉시 전송
- 페이지 이탈: `sendBeacon` 으로 flush

### 3-2. 사용자 식별 / CTA Funnel

#### 사용자 식별

```
사용자 식별자 = user_id (로그인 시) || device_id (비로그인)
```

- `device_id`: 클라이언트가 처음 진입 시 자체 생성한 UUID. 앱은 Keychain, 웹은 1st-party 쿠키 + localStorage 영속 저장
- **단순화 가정**: device_id 가 안정 발급. 현실에서는 앱 삭제/쿠키 초기화/크로스 디바이스로 영구 식별 불가 — 본 설계 범위 외
- 로그인 시 mapping 저장: `device_to_user:{device_id} = user_id` → 비로그인 행동을 user_id 에 attach (배치)

#### CTA Funnel: impression_id

핵심: **노출 → 클릭 → 장바구니** 를 같은 키로 추적.

```
추천 API 응답 시 → 각 상품에 impression_id (UUID) 발급
클라이언트가 모든 후속 이벤트에 같은 impression_id 포함
ClickHouse 에서 impression_id 로 join → funnel 분석
```

`impression_id` 유효기간 24h, 클라이언트 localStorage 보관, 서버 측 별도 저장 없음.

### 3-3. 추천 후보 흐름 + 캐싱

#### Redis 키 구조

```
user:{uid}:candidates       Sorted Set    L2: ML 후보, TTL 6h
user:{uid}:profile          Hash          L3: 사용자 행동 요약, TTL 30d
product:{pid}:related       Sorted Set    상품 연관성, TTL 24h
popular:overall             Sorted Set    인기 상품, TTL 1h
recommendations:{uid}       String        L1: 최종 응답 JSON, TTL 5m
```

#### 후보 생성 흐름

```
1. L2 후보 풀 (ML 모델 생성, 50개)
2. + recent_views 기반 연관 상품 (30개)
3. + popular fallback (부족 시 20개)
4. 선호 카테고리/브랜드 가중치 적용
5. recent_views[:20] dedup
6. 상위 10개 + impression_id 발급
7. L1 캐시 저장
```

Redis 호출 3~5 RTT, in-memory 처리 합쳐 ~30ms.

#### 3계층 캐시

| 계층 | 갱신 주체 | TTL | 진실 소스 |
|------|----------|-----|----------|
| L1: 응답 | 추천 API (요청 시) | 5m | — (재생성 가능) |
| L2: 후보 풀 | ML 파이프라인 (6h 배치) | 6h | ML 모델 |
| L3: 프로필 | 실시간 컨슈머 | 30d | Kafka 이벤트 |

**세 계층 모두 캐시**. Redis 장애 시 진실 소스에서 재구성 가능.

**Cache stampede 방지**: TTL jitter (5분 ± 1분 랜덤) 또는 single flight.

#### Redis 메모리 추정

활성 사용자 60만 (어제+오늘 방문) 기준. UUID 36 bytes 가정.

**전제값**:
- userId / productId: UUID 36 bytes (string)
- 상품 SKU: 10만개
- 추천 1건당 상품 10개
- 키당 Redis 오버헤드 (redisObject, sds 등): ~100 bytes

**L1: 추천 응답 캐시** `recommendations:{uid}` (String, JSON)

```
한 entry 내용:
  - 10개 item × ~160 bytes (product_id 36 + impression_id 36 + score + source + JSON 구조)
  = ~1.6 KB
한 entry 합계: key 50 + value 1,600 + 오버헤드 200 = ~2 KB

TTL 5분이라 활성 사용자 전체가 아닌 5분 윈도우 안 사용자만 살아있음:
  - 평균: 12 QPS × 300s = ~3,600명 → 7 MB
  - 피크: 300 QPS × 300s = ~90,000명 → 180 MB
```

**L2: 사용자별 후보 풀** `user:{uid}:candidates` (Sorted Set, 50 멤버)

```
멤버당: product_id 36 + score 8 + skip list 노드 오버헤드 40 = ~85 bytes
50 멤버: 50 × 85 = 4,250 bytes
한 entry: 키 50 + value 4,250 + 오버헤드 200 = ~4.5 KB

활성 사용자 60만 × 4.5 KB = ~2.7 GB
```

**L3: 사용자 프로필** `user:{uid}:profile` (Hash)

```
필드별 추정:
  - recent_views: 50개 product_id (UUID) → 50 × 36 = 1,800 bytes
  - recent_clicks: 30개 → 1,080 bytes
  - recent_carts: 10개 → 360 bytes
  - preferred_categories: 10개 → 200 bytes
  - preferred_brands: 10개 → 200 bytes
  - 메타 (timestamp 등) → 200 bytes
한 hash 값: ~4 KB
한 entry: 키 50 + value 4,000 + 오버헤드 200 = ~4.2 KB

활성 사용자 60만 × 4.2 KB = ~2.5 GB
```

**상품 연관 저장소** `product:{pid}:related` (Sorted Set, 20 멤버)

```
20 멤버 × 85 bytes = 1,700 bytes
한 entry: ~1.85 KB

상품 10만 SKU × 1.85 KB = ~185 MB
```

**인기 상품** `popular:overall` + `popular:category:{cat}`

```
overall: 10,000개 × 85 bytes = ~850 KB
카테고리별 (100 카테고리 × 1,000개): ~8.5 MB
합: ~10 MB
```

**합계**

| 항목 | 크기 |
|------|------|
| L1 응답 캐시 (피크) | ~180 MB |
| L2 후보 풀 | ~2.7 GB |
| L3 사용자 프로필 | ~2.5 GB |
| 상품 연관 | ~185 MB |
| 인기 상품 | ~10 MB |
| 기타 (sse_routing, quota 등) | ~50 MB |
| **이론 합계** | **~5.6 GB** |
| **+ Redis 오버헤드 (×2)** | **~11 GB** |
| **권장 인스턴스 (여유 30%)** | **15-20 GB** |

Redis 의 실제 메모리 사용량은 이론값의 1.5~2배가 일반적 (단편화, 패딩, 권장 사용률 70-80%).

**확장 시 메모리 변화**:
- DAU 2배 (활성 120만): L2 + L3 가 비례 증가 → ~10 GB → 권장 ~20 GB
- DAU 10배 (활성 600만): ~50 GB → 단일 인스턴스 한계 → Cluster 또는 용도별 분리

**UUID 대신 정수 ID 쓰면**: product_id / user_id 가 4-8 bytes 로 줄어 메모리 1/2 ~ 1/3 절약 가능. 단 UUID 의 무작위성/충돌 방지가 더 가치 있다면 그대로 유지.

#### HA 구성: Sentinel

- 데이터 15 GB 가 단일 인스턴스 한계 안 → Cluster 불필요
- Lua script, Pub/Sub 자유로움 (Cluster 의 slot 제약 없음)
- DAU 5배+ 증가 시 Cluster 또는 용도별 인스턴스 분리 검토

#### 신선도 (도메인 특성)

매일 신상 + 매일 방문 → 어제 본 상품 dedup 핵심:
- 후보 생성 시 `recent_views[:20]` 제외
- 노출 이력 추적 (`user:{uid}:impressed`, TTL 7d) → 가중치 감소
- 신상 boost 는 ML 모델 단

### 3-4. 추천 API 응답 + Fallback

#### 응답 흐름

```python
def get_recommendations(user_id):
    # L1 hit
    if cached := redis.get(f"recommendations:{user_id}"):
        return cached
    
    # 정상 경로
    try:
        candidates = generate_candidates(user_id)   # L2/L3
        top_10 = rank_and_select(user_id, candidates)
        redis.setex(f"recommendations:{user_id}", 300, top_10)
        return top_10
    except RedisError:
        # fallback
        return self.fallback_products[:10]
```

#### Fallback: In-memory 정적 인기 상품

**원칙**: fallback 은 **외부 의존이 0** 인 단일 경로. 다단계 fallback (Redis 외 DB, ML API 등) 은 동시 장애 가능성 때문에 신뢰성 낮음.

```python
class RecommendationAPI:
    def __init__(self):
        # 시작 시 S3 에서 인기 상품 top 100 로드
        self.fallback_products = load_from_s3()
        # 1시간마다 갱신 시도. 실패해도 기존 데이터 유지
        start_periodic_refresh(3600)
```

- Redis 장애 시: 메모리에서 즉시 응답 (개인화 0, 빈 응답 X)
- S3 도 죽으면: 패키지 내장 데이터 사용 (stale 하지만 절대 안 죽음)
- 트레이드오프: 모든 사용자에게 같은 추천 → CTA 큰 폭 하락 (불가피)

#### 저지연 보장

| 흐름 | 지연 |
|------|------|
| L1 hit | ~5ms |
| L2 miss → 후보 생성 | ~30ms |
| Redis 장애 → in-memory fallback | ~1ms |

**최적화**:
- Redis MGET / pipeline 으로 RTT 묶기
- 랭킹은 in-memory (외부 호출 없이)
- 응답은 product_id + 메타데이터만 (가벼움)

### 3-5. 대규모 트래픽 / 피크 대응

#### 피크 패턴

| 시간 | 부하 |
|------|------|
| 출/퇴근, 점심, 심야 | 평시 ×3 |
| 플랫폼 이벤트 (빅세일) | 평시 ×10~20 |

#### 일반 피크 대응

- **추천 API 수평 확장**: stateless. 큐 길이/CPU 기반 autoscaling
- **L1 캐시 hit ratio 극대화**: 같은 사용자 5분 재진입은 즉시 응답
- **이벤트 수집 분리**: 추천 API 와 별도 서버 → 부하 격리
- **클라이언트 배치**: impression 5초 단위 → 서버 부하 1/5

#### 플랫폼 이벤트 대응

평소 autoscaling 으로 못 따라잡음. 사전 대응:
- **사전 스케일링**: 이벤트 시작 전 인스턴스 증설
- **L1 TTL 임시 연장** (5m → 15m): 후보 생성 부하 감소
- **이벤트 샘플링**: impression 1/10 (click/cart 는 유지)
- **활성 사용자 캐시 워밍업**: 이벤트 시작 전 미리 준비

#### Hot Key

피크 시 인기 상품 키 (`product:{popular}:related`) 에 부하 집중:
- 추천 API in-memory 캐시 (1분 TTL)
- 또는 Redis 클러스터 환경에서 hot key 분산

---

## 4. 설계 장점

- **읽기/쓰기 파이프라인 완전 분리**: 추천 API 와 이벤트 수집이 독립. 부하 격리.
- **3계층 캐시 + 진실 소스 분리**: L1/L2/L3 모두 캐시. Redis 장애 시 재구성 가능. 최후엔 in-memory fallback 으로 응답 보장.
- **At-least-once + 저장소 dedup**: ClickHouse ReplacingMergeTree, Redis 멱등 자료구조로 자동 dedup. 별도 dedup 인프라 없이 effectively-exactly-once.
- **Hot/Cold Path 분리**: 실시간은 Redis (빠름), 분석/학습은 ClickHouse (영속). 책임 명확.
- **CTA Funnel = impression_id**: 노출-클릭-장바구니가 같은 키로 추적. A/B 테스팅도 같은 메커니즘.
- **도메인 특성 반영**: 어제 본 상품 dedup, 신상 boost.

---

## 5. 설계 단점 / 트레이드오프

- **Redis 의존도 큼**: L1/L2/L3 + 후보 저장소 모두 Redis. 장애 시 in-memory fallback 으로 응답은 보장하지만 **개인화 0, 모든 사용자 동일 추천 → CTA 큰 폭 하락**. Sentinel HA 필수.
- **L1 캐시 5분 TTL 의 trade-off**: 짧으면 신선도 ↑ 부하 ↑, 길면 반대. 신상 입고 후 최대 5분 반영 지연.
- **외부 ML 모델 의존**: 후보가 6시간 배치로 갱신 → 그 사이 신상/트렌드 반영 늦음. 보완은 연관 상품 + 인기 상품.
- **device_id 한계**: 자체 UUID 안정 발급 가정이지만 현실은 앱 삭제/쿠키 삭제로 영구 식별 불가. 로그인 유도가 개인화 품질에 직결.
- **CTA 지표 지연**: ClickHouse merge 가 background → 실시간 funnel 측정은 약간의 지연 (분 단위) 필요. 실시간 대시보드는 본 설계 범위 외.

### 메시지 유실 가능성

| 단계 | 가능성 | 영향 | 복구 |
|------|-------|------|------|
| 클라이언트 → API | ~0.05% | 이벤트 손실 | localStorage 버퍼링 |
| API → Kafka | ~0.01% | 손실 | acks=all 로 최소화 |
| Kafka → 컨슈머 | 0 | — | PEL 기반 |
| ClickHouse 적재 | 0 | — | offset commit after success |

합산 0.1% 이하 달성.

---

## 6. 마무리

### 개인적 의견

이 시스템에서 가장 중요한 통찰 두 가지:

**1. 읽기와 쓰기의 분리**: 추천 응답 (저지연 + 캐시 친화) 과 이벤트 수집 (대용량 + 유실 방지) 은 SLA 와 부하 패턴이 완전히 다름. 같은 서버에 묶으려는 유혹을 피하고 처음부터 분리해야 양쪽 다 잘 동작.

**2. fallback 은 외부 의존 0 이어야 진짜 fallback**: Redis 가 죽을 정도의 장애에선 DB/ML API 도 동시 장애 가능성이 큼. 진짜 최후 보루는 추천 API 인스턴스의 메모리. 개인화는 사라지지만 빈 응답은 막을 수 있음.

또한 **dedup 은 저장소 책임**으로 위임하는 게 단순함. 컨슈머 코드 안에서 멱등 처리 신경 쓰는 것보다 ClickHouse ReplacingMergeTree 같은 기능 활용이 훨씬 KISS.

### 추가 고려

- **Feature Store** (Feast 등) 로 프로필 + 상품 메타 통합 관리
- **실시간 ML inference** 도입 시 신선도 ↑ (현재는 6h 배치)
- **CDC**: 상품 정보 변경 → Kafka → 추천 후보 자동 무효화
- **개인정보**: 행동 데이터 보관 기간, 동의, 삭제 요청

### 참고

- 네이버 deview "실시간 추천 시스템을 위한 Feature Store 구현기"
- 무신사 / 29CM 의 추천 시스템 사례