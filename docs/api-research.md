# Runners Plan — 피트니스 플랫폼 API 사전 조사

> 조사일: 2026-07-09
> 목적: Strava / Garmin Connect / 삼성헬스 / 애플워치(HealthKit) 계정을 연결해
> **"지난달 달린 km"** 를 가져올 수 있는지 확인.

## 요약 (결론 먼저)

| 플랫폼 | 지난달 러닝 km 조회 | 연결 방식 | 서버(웹)만으로 가능? | 주요 제약 |
|---|---|---|---|---|
| **Strava** | ✅ 가능 | OAuth 2.0 (웹) | **✅ 유일하게 가능** | 개발자 구독 $11.99/월(2026-06-30~), 10명 초과 시 앱 심사, AI/ML 사용 금지 조항 |
| **Garmin Connect** | ⚠️ 기술적으로 가능 / 현실적으로 막힘 | OAuth 2.0 PKCE + Webhook | 가능 (승인 시) | **신규 개발자 프로그램 접수 중단(2026 중반, 재개 미정)** |
| **삼성헬스** | ✅ 가능 (Android 앱 필요) | Health Connect SDK (온디바이스) | ❌ 불가 | 웹 API 없음(파트너 전용), Health Connect 30일 히스토리 제한 → history 권한 필요 |
| **애플워치** | ✅ 가능 (iOS 앱 필요) | HealthKit (온디바이스) | ❌ 불가 | 웹/REST API 없음, Apple Developer Program $99/년 |

**핵심 시사점**
- 순수 웹 서비스(백엔드 OAuth 연동)만으로 가능한 건 **Strava뿐**.
- 삼성헬스·애플워치는 사용자 기기에서 도는 **네이티브 앱(Android/iOS)이 필수** — 앱이 기기에서 데이터를 읽어 백엔드로 업로드하는 구조.
- Garmin은 공식 API가 훌륭하지만(백필로 5년 과거 조회) 현재 **신규 API 키 발급이 불가**. 대안: (a) 사용자에게 Garmin→Strava 자동 동기화를 켜게 유도, (b) Terra 등 유료 애그리게이터.
- MVP 권장 순서: **Strava 먼저** → Android(Health Connect: 삼성헬스+가민 일부 커버) → iOS(HealthKit) → Garmin 공식(프로그램 재개 시).

---

## 1. Strava — ✅ 가능 (웹 OAuth, 가장 쉬움)

- **인증**: 표준 OAuth 2.0. `https://www.strava.com/oauth/authorize` → 코드 교환. 액세스 토큰 6시간 만료, 리프레시 토큰 제공.
- **스코프**: `activity:read` (전체공개/팔로워 활동), **`activity:read_all`** (비공개 러닝 포함 — 정확한 월간 합계엔 이게 필요. 없으면 비공개 런이 조용히 누락됨).
- **지난달 km 계산**: `GET /api/v3/athlete/activities?after=<지난달1일 epoch>&before=<이번달1일 epoch>&per_page=200`
  → 응답에서 `sport_type == "Run"` 필터링(서버측 타입 필터 없음) → `distance`(미터) 합산 ÷ 1000.
  - 주의: `GET /athletes/{id}/stats`의 `recent_run_totals`는 **달력 월이 아니라 최근 4주 롤링**이라 부적합.
- **레이트 리밋**: 앱 전체 200req/15분, 2,000/일 (읽기 100/15분, 1,000/일).
- **승인/비용**:
  - Standard 티어: 즉시 사용 가능하나 **연결 사용자 10명 제한**. 앱 심사 통과 시 9,999명.
  - **2026-06-30부터 Standard 티어 이용에 개발자 본인의 Strava 구독($11.99/월) 필요.**
  - Extended Access(1만+ 사용자): Strava 승인 필요, 구독 불요.
- **약관 주의 (2024-11 개정)**:
  - API 데이터는 **해당 사용자 본인에게만 표시** 가능 (공개 리더보드 등 금지). 본인 월간 km 표시는 문제 없음.
  - **API 데이터를 AI/ML 모델에 사용 금지** — LLM 기반 훈련 플랜 생성에 Strava 데이터를 넣는 것은 법적 회색지대. 설계 시 검토 필요.
  - 사용자 연결 해제 시 데이터 즉시 삭제 의무 (deauthorization webhook 처리).

## 2. Garmin Connect — ⚠️ 기술적 가능 / 신규 발급 중단

- **인증**: OAuth 2.0 PKCE (OAuth 1.0a는 2026-12-31 종료). 사용자 키는 토큰이 아닌 Garmin User ID 사용.
- **데이터 모델**: REST 쿼리형이 아니라 **Webhook(Ping/Push) 기반**. 임의 조회 엔드포인트 없음.
  - 과거 데이터는 **Backfill API**: 요청당 최대 90일 범위, Activity API 기준 **최대 5년 전까지** 소급 가능. 비동기(요청 → 잠시 후 webhook으로 도착) → "연결 직후 바로 표시" UX에 지연 고려 필요.
  - Garmin 측 데이터 보존 ~7일 → 수신 즉시 자체 DB에 저장 필수.
- **러닝 거리**: Activity Summaries의 `activityType`(RUNNING + 트레드밀/트레일/인도어/버추얼 등 서브타입 포함해야 함) + `distanceInMeters` 합산.
- **승인/비용**: 무료이나 **기업(비즈니스) 대상 프로그램**. 개인/취미 개발자 불가.
  - **현재(2026 중반) 신규 신청 접수 중단** — 신청 폼 제거, 재개 일정 없음. 기존 파트너만 유지.
- **비공식 라이브러리** (`python-garminconnect` 등): 사용자 비밀번호 직접 로그인 방식 → ToS 위반 + 2026-03 Cloudflare 봇 차단으로 수시 파손. 다중 사용자 프로덕션에는 부적합.
- **현실적 대안**:
  1. 사용자에게 Garmin→Strava 자동 동기화를 켜게 안내하고 Strava API로 수집 (가장 저렴).
  2. Garmin 프로덕션 키를 이미 보유한 유료 애그리게이터(Terra, Spike, Open Wearables 등) 경유.

## 3. 삼성헬스 — ✅ 가능하나 Android 앱 필수

- **공개 웹/REST API 없음.** REST API(`data-api.samsunghealth.com`)가 존재하지만 문서 자체가 "partner-only" — 공개 신청 경로 없음.
- **경로 A (권장): Android Health Connect**
  - 삼성헬스는 Health Connect와 양방향 동기화(운동 세션·거리 포함, v6.22.5+).
  - 우리 Android 앱이 Health Connect Jetpack SDK로 `ExerciseSessionRecord`(러닝) + `DistanceRecord` 집계(`AggregateRequest` + `TimeRangeFilter.between(지난달1일, 이번달1일)`).
  - **30일 히스토리 제한**: 권한 최초 부여 시점 기준 30일 이전 데이터는 기본 차단 → "지난달"(최대 ~40일 전) 조회에는 **`PERMISSION_READ_HEALTH_DATA_HISTORY` 권한 필수**.
  - **백필 함정**: 삼성헬스는 사용자가 Health Connect 연동을 켠 시점 이후 데이터만 공유 — 신규 사용자는 지난달 데이터가 Health Connect에 없을 수 있음(사용자가 삼성헬스 설정에서 동기화를 켜야 함).
  - Google Play 배포 시 Health apps declaration 심사 필요. 삼성 승인은 불필요.
- **경로 B: Samsung Health Data SDK** (온디바이스, 2025-07 GA / 구 SDK는 deprecated)
  - 삼성헬스 로컬 저장소를 직접 조회 — 30일 제한 없이 전체 이력 접근, 거리 포함 운동 세션 제공.
  - 개발/테스트는 개발자 모드로 무승인 가능하나 **스토어 배포에는 삼성 파트너 승인 필요**.
- **웹 전용으로는 불가.** 애그리게이터를 써도 그 SDK를 넣은 Android 앱이 기기에 있어야 함.

## 4. 애플워치 (Apple Health / HealthKit) — ✅ 가능하나 iOS 앱 필수

- **웹/REST API·OAuth 없음** (확정). Health 데이터는 기기에만 존재 → **iOS 컴패니언 앱이 유일한 경로**.
- **지난달 러닝 km**: `HKWorkout` + `predicateForWorkouts(with: .running)` + 지난달 날짜 범위 predicate → 워크아웃별 `statistics(for: HKQuantityType(.distanceWalkingRunning))?.sumQuantity()` 합산.
  - `totalDistance`는 iOS 16+ deprecated.
  - `distanceWalkingRunning` 통계 쿼리 단독 사용 금지(걷기 포함됨) — 워크아웃 단위로 합산할 것.
  - **과거 조회 제한 없음** — 워치 기록은 아이폰 Health 스토어에 동기화되어 수개월~수년치 조회 가능. 최초 연결 시 지난달 백필 문제 없음.
- **요건**: Apple Developer Program $99/년, HealthKit entitlement, `NSHealthShareUsageDescription`, 데이터 타입별 사용자 권한 승인.
- **서버 전송**: 허용되나 App Store 심사 지침 5.1.3 준수 — 사용자 동의·개인정보처리방침 필수, 건강 데이터의 광고/마케팅/판매 금지, iCloud 저장 금지.
- **UX 함정**: 읽기 권한 거부 여부를 API로 알 수 없음 — 거부 시 그냥 빈 결과(= "지난달 안 뜀"과 구분 불가).

## 5. 참고: 애그리게이터 (직접 연동 대신 유료 우회)

Terra / Rook / Spike / Thryve / Validic 등 — Strava·Garmin은 서버측 연동 제공, 삼성/애플은 여전히 자사 SDK를 넣은 모바일 앱 필요.
비용 감: 대략 **연결 사용자당 월 $0.5~2** (Terra는 연 결제 기준 월 ~$399부터). Garmin 공식 프로그램이 막힌 동안 유일한 합법적 Garmin 경로.

## 6. 제안 로드맵

1. **Phase 1 (웹만으로 시작)**: Strava OAuth 연동 → 지난달 km 표시. Garmin 사용자는 Garmin→Strava 동기화 안내로 커버.
2. **Phase 2 (Android 앱)**: Health Connect 연동 → 삼성헬스(+Health Connect에 쓰는 타 앱) 커버. history 권한 + Play 심사 준비.
3. **Phase 3 (iOS 앱)**: HealthKit 연동 → 애플워치 커버. $99/년 + 5.1.3 준수.
4. **Phase 4 (선택)**: Garmin 공식 프로그램 재개 모니터링 또는 애그리게이터 계약.

주의: Strava 데이터의 AI/ML 사용 금지 조항 — LLM으로 훈련 플랜을 생성할 계획이라면 Strava 데이터 투입 여부를 법적으로 재검토할 것.
