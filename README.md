# 마주한끼 (Maju-Hankki) — 식사 동반자 매칭 서비스

> 부트캠프 2차 프로젝트 (2인 팀) · 혼자 밥 먹기 싫은 순간, 근처에서 같은 시간에 밥 먹을 사람을 실시간으로 매칭해주는 서비스

이 저장소는 팀 레포(fork)를 기반으로, 본인(sangmin)이 담당한 백엔드 개발 전반을 중심으로 정리한 포트폴리오용 README입니다. 실제 커밋 이력과 코드, docs/troubleshooting 문서를 근거로 작성했습니다.

## 한눈에 보기

| 항목 | 내용 |
| --- | --- |
| 서비스 | 식사 동반자 실시간 매칭 (시간·장소·음식 성향 기반) |
| 팀 구성 | 2인 (부트캠프 2차 프로젝트) |
| 담당 역할 | 백엔드 개발 전반 |
| 차별점 | 자유 텍스트 식사 성향을 임베딩으로 변환해 호환도까지 점수화 |
| 대표 트러블슈팅 | 리프레시 토큰 쿠키 중복 발급 이슈 해결 |

```mermaid
flowchart LR
    A[사용자] -->|시간/장소/성향 입력| B[매칭 요청]
    B --> C{PostGIS 반경 필터}
    C --> D[pgvector 호환도 점수 계산]
    D --> E{상호 수락?}
    E -- Yes --> F[매칭 확정 + 채팅방 생성]
    E -- No --> B
    F --> G[WebSocket/STOMP 1:1 채팅]
```

## 담당 역할 (sangmin — 백엔드 개발 전반)

| 영역 | 작업 내용 |
| --- | --- |
| 인증/인가 | Spring Security + JWT 구조 설계·구현 |
| 소셜 로그인 | Google OAuth2 연동 |
| 실시간 채팅 | WebSocket/STOMP 기반 1:1 채팅 |
| 지도 연동 | 선호 지역(시/군/구) 조회 API, 카카오맵 마커 UI 개선 |
| 매칭 점수 | pgvector 단어 단위 임베딩 교차 비교로 호환도 산출 로직 리팩터링(1NF 데이터 모델, N×M 비교, myScore 분리) |
| 트러블슈팅 | 리프레시 토큰 쿠키 중복 발급 이슈 해결 |

## 기술 스택

![Java](https://img.shields.io/badge/Java-007396?style=for-the-badge&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring%20Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring%20Data%20JPA-6DB33F?style=for-the-badge&logo=spring&logoColor=white)
![WebSocket](https://img.shields.io/badge/WebSocket%20%2F%20STOMP-010101?style=for-the-badge&logo=socketdotio&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![pgvector](https://img.shields.io/badge/pgvector-336791?style=for-the-badge&logo=postgresql&logoColor=white)
![PostGIS](https://img.shields.io/badge/PostGIS-008000?style=for-the-badge&logo=postgresql&logoColor=white)
![Kakao Maps](https://img.shields.io/badge/Kakao%20Maps%20API-FFCD00?style=for-the-badge&logo=kakao&logoColor=black)
![Google OAuth2](https://img.shields.io/badge/Google%20OAuth2-4285F4?style=for-the-badge&logo=google&logoColor=white)

> Redis 기반 실시간 매칭 대기열, Prometheus/Grafana 모니터링, Docker 인프라는 팀원이
> 구축한 영역이라 이 README의 "기술 스택"에서는 본인이 직접 다룬 것만 남겼다
> (PostGIS는 `Region` 엔티티의 좌표 컬럼에 한해 직접 사용했다).

## 핵심 구현 내용

### 1. Spring Security + JWT 인증 구조

| 구성요소 | 역할 |
| --- | --- |
| `JwtProvider` | 사용자 UUID·역할(UserRole)을 클레임에 담아 HS256 서명 Access Token 발급, issuer/audience 검증 |
| `JwtFilter` | 매 요청 토큰 검증 후 SecurityContext에 인증 정보 세팅 |
| `RefreshTokenRepository` + `RefreshTokenHasher` | Refresh Token을 해시로 DB 저장, `OpaqueTokenGenerator`로 발급해 Access Token과 분리 관리 |
| RTR(Refresh Token Rotation) | 재발급마다 토큰 교체, 재사용 감지(`RefreshTokenReuseDetectedException`) 시 세션 계열 전체 폐기 |
| `DelegatingPasswordEncoder` | bcrypt/argon2 선택적 적용 |
| CSRF | `CookieCsrfTokenRepository` + `SpaCsrfTokenRequestHandler`로 SPA에 맞게 처리 |

### 2. Google OAuth2 소셜 로그인

| 처리 흐름 | 내용 |
| --- | --- |
| 회원 조회 | `GoogleOAuthUserService`에서 Google `providerId`/이메일로 기존 회원 조회, 없으면 신규 가입 |
| 중복 가입 방지 | 동일 이메일로 다른 로그인 방식(LOCAL/KAKAO 등) 가입돼 있으면 차단 (`AUTH_005`) |
| 닉네임 충돌 처리 | 비어있거나 중복되면 `사용자_xxxxxxxx` 형태로 안전 생성 |
| 비활성 계정 차단 | 탈퇴 등으로 비활성화된 계정은 로그인 시점에 차단 (`AUTH_001`) |

### 3. WebSocket/STOMP 기반 실시간 채팅

| 구성요소 | 역할 |
| --- | --- |
| `WebSocketConfig` | `/ws-chat` STOMP 엔드포인트 등록 |
| `WebSocketHandshakeInterceptor` | 핸드셰이크 단계 쿠키 기반 JWT 인증 |
| `ChatSubscriptionInterceptor`(인바운드) / `ChatOutboundAccessInterceptor`(아웃바운드) | CONNECT/SUBSCRIBE/SEND 시점마다 채팅방 접근 권한 검증 |
| 악용 방지 | 메시지 크기 16KB 제한, 전송 버퍼/타임아웃 제한, `ChatMessageRateLimiter`로 전송 속도 제한 |
| 영속화 | `ChatMessage`, `ChatRoom` 엔티티 |

### 4. 선호 지역 조회 API + 카카오맵 마커 UI

| 구성요소 | 역할 |
| --- | --- |
| `RegionController` / `RegionRepository` | 지원하는 구 단위 행정구역 목록과 지도 대표 좌표(위도/경도)를 조회하는 API 제공 |
| 프론트엔드 (`match-map.js`) | 선호 지역을 시/군/구 select 방식으로 등록하도록 UX 개편, 카카오맵 마커를 화면 가운데 고정하는 형식으로 수정 |

> 좌표가 실제 행정동/법정동 코드 안에 있는지 서버에서 재검증하는 `KakaoRegionPinValidator`는
> 같은 팀 백엔드 파트인 region 검증 로직으로, 본인이 작성한 코드는 아니라 이 README에서는 제외했다.

### 5. pgvector 임베딩 기반 매칭 호환도 점수 계산

가장 공을 들인 부분입니다. 단순 태그 매칭만으로는 어떤 음식을, 어떤 분위기로 먹고 싶은지처럼 사용자가 자유 텍스트로 적는 미묘한 취향 차이를 반영할 수 없어서, 텍스트 임베딩 기반 의미 유사도를 도입했습니다.

```mermaid
flowchart TD
    A[PostGIS 반경 + 시간차 30분 1차 필터] --> B[신청자 희망 키워드 N개]
    A --> C[후보자 등록 키워드 M개]
    B --> D[N x M 전수 교차 비교]
    C --> D
    D --> E[코사인 유사도 3단계 스케일링]
    E --> F[고정 태그 80% + 임베딩 20% 가중 평균]
    F --> G[양방향 개별 점수 myScore]
```

| 설계 포인트 | 내용 |
| --- | --- |
| 데이터 모델 | `UserPersonalityEmbedding`을 1사용자-N단어(원자 레코드) 구조로 리팩터링(1NF 정규화). 단어 1개당 1536차원 벡터 1개를 독립 레코드로 저장 |
| 벡터 저장 | `@JdbcTypeCode(SqlTypes.VECTOR)` + `columnDefinition = "vector(1536)"`로 `float[]`를 PostgreSQL `vector(1536)` 컬럼에 직접 바인딩 (hibernate-vector), 임베딩 생성은 Spring AI(`spring-ai-starter-model-google-genai-embedding`) |
| 비교 방식 | `PersonalityCompatibilityCalculator.calculateFullMatrixCrossScore`로 N×M 전수 교차 비교 — 각 신청자 단어마다 가장 유사한 후보 단어 점수를 구해 평균 |
| 0점 처리 | 일치 태그 0개일 때 과거 50점(중립) 기본값을 버리고 0점으로 명확히 계산, 애매한 고점수 방지 |
| 최종 점수 | 고정 태그(80%) + 임베딩 유사도(20%) 가중 평균, 한쪽 신호만 있어도 있는 신호로 산출 |
| 양방향 점수 | `myScore` — 두 사용자 점수 중 낮은 쪽(pairScore) 대표값 방식을 버리고, 내가 상대를 원하는 정도/상대가 나를 원하는 정도를 각각 독립 계산 |
| 사전 필터 | PostGIS 공간 쿼리로 반경·시간차 30분 이내 후보를 1차로 좁힌 뒤에만 호환도 계산을 수행하도록 연동 (후보 탐색 자체는 팀원이 구현한 `BidirectionalCandidateSearchService`) |

```java
@JdbcTypeCode(SqlTypes.VECTOR)
@Column(nullable = false, columnDefinition = "vector(1536)")
private float[] embedding;
```

**코사인 유사도 3단계 스케일링** — 단순 코사인 유사도(-1~1)를 그대로 쓰면 실제 임베딩 공간에서 값이 좁은 범위(대략 0~0.5)에 몰려 변별력이 떨어지는 문제가 있어, 구간별로 다른 기울기를 적용했다.

| 코사인 유사도 구간 | 점수 구간 | 의미 |
| --- | --- | --- |
| 0 ~ 0.20 | 0 ~ 30점 | 거의 무관한 단어 |
| 0.20 ~ 0.50 | 30 ~ 70점 | 느슨하게 관련 |
| 0.50 ~ 1.0 | 70 ~ 100점 | 의미적으로 가까운 단어 |

## 트러블슈팅

### 1. Refresh Token 쿠키 중복 발급 및 세션 꼬임

| 구분 | 내용 |
| --- | --- |
| 증상 | 로그인/재발급 시 동일 이름(`refreshToken`) 쿠키가 2개 생성 → 요청마다 두 값이 함께 전송 → 토큰 회전(RTR) 중 어떤 쿠키가 유효한지 불명확해져 정상 세션이 강제 폐기되거나 401 반복 |
| 원인 | 초기 쿠키 경로가 `Path=/auth`였다가 `Path=/`로 변경됨. 브라우저는 쿠키를 이름+경로 조합으로 식별해 새 쿠키가 기존 쿠키를 덮어쓰지 않고 공존 → `/auth` 하위 요청 시 두 경로 모두 매칭되어 쿠키 2개 동시 전송 |
| 해결 1 | `AuthCookieUtil.deleteLegacyRefreshTokenCookie()` 추가 — 토큰 발급/재발급/로그아웃 시 `Path=/auth, Max-Age=0` 쿠키를 함께 내려 구버전 쿠키를 명시적으로 만료 |
| 해결 2 | `AuthController`에서 요청에 담긴 모든 `refreshToken` 쿠키를 스트림으로 수집 후 `distinct()`로 중복 제거 → 레거시 DB 세션까지 안전하게 일괄 폐기 |

## 실행 방법

```bash
# 1. 인프라 (Redis / Prometheus / Grafana) 기동
docker compose up -d

# 2. 백엔드 (Spring Boot)
./gradlew bootRun

# 3. 프론트엔드
cd frontend
npm install
npm run dev
```

PostgreSQL은 pgvector, PostGIS(hibernate-spatial) 확장이 활성화되어 있어야 하며, Google OAuth2 / Kakao REST API 키 등은 환경 변수 또는 application.yml 설정이 필요합니다.
