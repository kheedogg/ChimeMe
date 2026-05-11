# ChimeMe 개발자 기술 검토서

> **작성자**: Senior Developer (Architect Review)
> **작성일**: 2026-05-12
> **대상 규모**: 5,000 동시 사용자 (DAU 기준)
> **스택**: React Native + Expo + Firebase (Auth / Firestore / Storage / FCM)

---

## 1. 데이터 모델 검토 (Firestore)

### 1.1 현재 컬렉션 구조 (spec 기반 재구성)

| 컬렉션 경로 | 문서 키 | 주요 필드 | 근거 |
|---|---|---|---|
| `users/{uid}` | Firebase Auth UID | `displayName`, `photoUrl`, `phoneNumber`, `createdAt` | `spec/onboarding/overview.md:18`, `spec/auth-verify/overview.md:15` |
| `groups/{groupId}` | 자동 생성 ID | `name`, `ownerId`, `memberIds[]`, `lastActivityAt`, `createdAt` | `spec/main/data.md:7-9` |
| `groups/{groupId}/posts` | 자동 생성 ID | `authorId`, `videoUrl`, `hourSlot`, `createdAt` | `spec/group-detail/data.md:14-18` |
| `memberships/{uid}_{groupId}` | 복합 키 | `unreadVideoCount`, `joinedAt`, `role` | `spec/main/data.md:16` |

### 1.2 인덱스 설계

현재 기획서에서 명시된 인덱스:
- `groups`: `memberIds array_contains` + `lastActivityAt desc` 복합 인덱스 (`spec/main/data.md:35`)
- `groups/{groupId}/posts`: `hourSlot desc` 단일 필드 인덱스 (`spec/group-detail/data.md:50`)
- (선택) `authorId + hourSlot` 복합 인덱스 (`spec/group-detail/data.md:51`)

**문제점 및 권고:**

| # | 이슈 | 설명 | 권고 |
|---|---|---|---|
| 1 | `memberIds` array-contains 한계 | Firestore `array-contains`는 **쿼리당 1개**만 사용 가능. 현재는 문제없으나 향후 필터 조건 추가 시 제약 | 현재 구조 유지. 추가 필터 필요 시 `memberships` 컬렉션 기반 쿼리로 전환 |
| 2 | `groups/{groupId}` 핫 도큐먼트 | `lastActivityAt`을 매 영상 업로드마다 갱신 → 그룹 8명이 동시에 업로드 시 **1초 1회 쓰기 제한**에 근접. 5,000 DAU / 평균 3개 그룹 = ~1,800 활성 그룹, 정각 직후 집중 쓰기 발생 | **Cloud Function에서 배치 갱신** 또는 `lastActivityAt`을 클라이언트에서 직접 쓰지 않고 Cloud Function이 post 생성 트리거로 비동기 갱신 |
| 3 | `memberships` unreadVideoCount 핫 쓰기 | 새 영상 업로드 시 Cloud Function이 그룹 멤버 7명의 `memberships` 문서에 각각 +1 쓰기 → 정각 직후 8명 x 7 = 56회 쓰기/그룹. 1,800 활성 그룹이면 ~100,800회 쓰기 burst | **분산 카운터 패턴** 적용 또는 `unreadVideoCount` 대신 클라이언트에서 마지막 읽은 시간 기반 계산으로 전환 (쓰기 → 읽기로 부하 전환) |
| 4 | posts 서브컬렉션 vs 최상위 컬렉션 | 현재 `groups/{gid}/posts`로 서브컬렉션 설계. 장점: 보안 룰이 간결, 그룹 단위 쿼리 자연스러움. 단점: 컬렉션 그룹 쿼리 필요 시 추가 인덱스 | 현재 구조 유지 권고. 7일 보존이므로 문서 수 관리 가능 |

### 1.3 시간슬롯 영상 도큐먼트 모델 권고

기획서의 `spec/group-detail/data.md:14-18`에서는 `posts` 서브컬렉션에 flat하게 저장. 그러나 **시간 슬롯별 그룹핑**이 핵심 접근 패턴이므로 다음 구조를 권고:

```
groups/{gid}/slots/{YYYYMMDDHH}          ← 슬롯 메타 문서 (선택)
groups/{gid}/slots/{YYYYMMDDHH}/clips/{uid}  ← 개별 영상
```

**장점:**
- 슬롯 단위 쿼리가 `collectionGroup` 없이 직접 접근 가능
- 멤버별 영상 존재 여부를 문서 존재로 판단 (쿼리 불필요)
- 보안 룰에서 `slots/{slotId}`의 시간 범위 검증 용이

**단점:**
- 기존 flat 구조 대비 경로가 깊어짐 (4단계)
- 빈 슬롯 메타 문서 관리 필요

**트레이드오프**: 현재 flat 구조도 5,000 DAU에서는 동작하나, 슬롯 기반 접근이 **매우 빈번**하므로 nested 구조가 읽기 효율에서 우위.

### 1.4 채팅 메시지 모델

| 옵션 | 구조 | 장점 | 단점 |
|---|---|---|---|
| A. Firestore 서브컬렉션 | `groups/{gid}/messages/{msgId}` | 통합 관리, 보안 룰 일원화, 추가 비용 없음 | Firestore 실시간 리스너 비용 (메시지 수 x 활성 리스너), 타이핑 인디케이터 어려움 |
| B. 외부 채팅 SDK (SendBird, Stream) | 별도 서비스 | 읽음 표시/타이핑/페이지네이션 내장, Firestore 부하 분리 | 추가 비용 ($0.01~0.02/MAU), 인증 연동 복잡성 |
| C. Firebase Realtime Database | `chats/{gid}/messages/{msgId}` | Firestore보다 실시간 특화, 비용 저렴 (대역폭 기반) | 쿼리 능력 제한, 두 DB 관리 부담 |

**권고**: 5,000 DAU 규모에서는 **옵션 A (Firestore 서브컬렉션)**로 시작. 그룹당 최대 8명이므로 동시 리스너 부하 관리 가능. 다만 `spec/chat/overview.md:22`의 "모두 읽음" 구현 시 **각 멤버의 lastReadAt을 그룹 문서 또는 memberships에 저장**해야 하며, 이것이 핫 도큐먼트 문제를 악화시킬 수 있음.

**"모두 읽음" 구현 권고:**
```
memberships/{uid}_{groupId}.lastReadMessageAt: Timestamp
```
클라이언트가 채팅 진입 시 갱신. "모두 읽음" 판정은 클라이언트에서 `memberships` 문서 8개를 읽어 비교 (쓰기 fan-out 대신 읽기 fan-in).

---

## 2. Storage / 영상 처리

### 2.1 영상 코덱/해상도 권고

| 항목 | 권고값 | 근거 |
|---|---|---|
| 코덱 | H.264 (AVC) | HEVC는 Android 호환성 불균일 (API 24+ 부분 지원). H.264는 범용 디코딩 보장 |
| 해상도 | 720p (1280x720) | 3초 영상에 1080p는 과잉. 720p로도 모바일 화면에서 충분한 품질 |
| 비트레이트 | 2~3 Mbps | 720p H.264 기준 적정. 3초 기준 0.75~1.125 MB/클립 |
| 컨테이너 | MP4 | 범용 호환성, 스트리밍 가능 (`moov` atom 앞 배치) |
| 프레임레이트 | 30fps | 일상 영상에 60fps 불필요 |

### 2.2 용량 및 비용 추정

**가정:**
- 1인당 평균 3개 그룹 가입
- 활성 시간: 하루 12시간 (오전 8시~오후 8시)
- 업로드율: 활성 슬롯의 50% (12슬롯 x 50% = 6회/일/인)
- 클립 크기: 1 MB (720p H.264 2.5Mbps x 3초)

| 항목 | 산출 | 값 |
|---|---|---|
| 1인 일일 업로드 | 6 클립 x 1 MB | 6 MB |
| 1인 월간 업로드 | 6 MB x 30일 | 180 MB |
| 5,000 DAU 일일 총 업로드 | 5,000 x 6 MB | 30 GB/일 |
| 5,000 DAU 월간 총 업로드 | 30 GB x 30 | 900 GB/월 |
| 7일 보존 시 최대 저장량 | 30 GB x 7 | 210 GB |
| 월간 다운로드 (가정: 업로드의 3배 조회) | 900 GB x 3 | 2,700 GB |

**Firebase Storage 월 비용 추정 (Blaze Plan):**

| 항목 | 단가 | 월 비용 |
|---|---|---|
| 저장 (210 GB 평균) | $0.026/GB | ~$5.46 |
| 다운로드 (2,700 GB) | $0.12/GB | ~$324 |
| 업로드 (900 GB) | 무료 | $0 |
| **합계** | | **~$330/월** |

**경고**: 다운로드 비용이 지배적. CDN 캐시 적용 시 실제 origin 전송량을 60~70% 절감 가능 → ~$100~130/월.

### 2.3 썸네일 정책

| 항목 | 권고 |
|---|---|
| 생성 시점 | Cloud Function 트리거 (Storage `finalize` 이벤트) |
| 해상도 | 360x360 (정사각형, 그룹 카드용) + 720x720 (상세 미리보기용) |
| 포맷 | WebP (iOS 14+/Android 지원, JPEG 대비 30% 절감) |
| 저장 경로 | `thumbs/{groupId}/{slotId}/{uid}_360.webp` |
| CDN 캐시 | `Cache-Control: public, max-age=604800` (7일 = 보존 기간과 동일) |

### 2.4 자동 삭제 정책

- **Cloud Storage Lifecycle Rule**: `age: 8` (7일 보존 + 1일 버퍼) 적용하여 객체 자동 삭제
- **Firestore 문서 정리**: Cloud Scheduler (매일 03:00 UTC) → Cloud Function이 7일 이전 `slots/clips` 문서 배치 삭제 (500개 단위 batch)
- **주의**: Storage 객체 삭제와 Firestore 문서 삭제를 **별도 파이프라인**으로 운영. Storage lifecycle이 실패해도 Firestore 참조가 남아 깨진 링크 발생 → Function에서 양쪽 모두 정리

### 2.5 업로드 실패/재시도/리줌 정책

`spec/capture/overview.md:27-29` 기반:

| 항목 | 권고 |
|---|---|
| 재시도 전략 | 지수 백오프 (1s → 2s → 4s → 8s → 16s), 최대 5회 |
| 리줌 | Firebase Storage `uploadBytesResumable` 사용. 1 MB 클립이므로 리줌보다 재전송이 효율적일 수 있으나, 불안정 네트워크 대비 리줌 유지 |
| 마감 처리 | 슬롯 마감(다음 정각) 도달 시 큐에서 제거 + 로컬 알림 "업로드 시간이 지났어요" |
| 오프라인 큐 | `expo-task-manager` + AsyncStorage에 큐 상태 영속화. 앱 재시작 시 큐 복원 |
| 중복 방지 | 업로드 경로를 `{groupId}/{slotId}/{uid}.mp4`로 고정 → 동일 슬롯 재업로드 시 덮어쓰기 |

---

## 3. 시간 동기화 (매시간 마감 핵심)

### 3.1 기준 시간대

| 옵션 | 장점 | 단점 |
|---|---|---|
| A. 서버 UTC 단일 | 구현 단순, 글로벌 일관성 | 사용자에게 "09:00 슬롯"이 실제 로컬 09:00가 아닐 수 있음 |
| B. 사용자 프로필 timezone | 사용자 기대 시간과 일치 | 같은 그룹 내 다른 시간대 멤버 간 슬롯 불일치 |
| C. 디바이스 로컬 시간 | 별도 설정 불필요 | 시간대 변경 시 슬롯 중복/누락, 조작 가능 |

**권고: 옵션 B (사용자 프로필 timezone)**

- `users/{uid}.timezone` 필드 추가 (IANA 형식, 예: `Asia/Seoul`)
- 슬롯 키: UTC 기반 `YYYYMMDDHH_UTC`로 서버 저장, **표시만 로컬 변환**
- 이렇게 하면 같은 그룹의 서울 사용자와 도쿄 사용자가 동일 UTC 슬롯을 공유하면서 각자 로컬 시간으로 표시

**중요**: `spec/group-detail/data.md:24-26`에서 "타임존: 사용자 로컬"이라고 명시되어 있으나, 이것이 **저장**까지 로컬이면 그룹 내 시간대 혼재 시 치명적 버그 발생. 반드시 **저장은 UTC, 표시는 로컬** 원칙 확립 필요.

### 3.2 클럭 스큐 허용 윈도우

| 항목 | 권고값 | 근거 |
|---|---|---|
| 업로드 허용 윈도우 | 정각 + 0분 ~ +59분 59초 + **30초 grace** | 네트워크 지연 + 디바이스 클럭 오차 허용 |
| 서버 측 검증 | Cloud Function 또는 Security Rule에서 `request.time` 기준 판정 | 디바이스 시간 조작 방지 |
| NTP 권고 | 앱 시작 시 서버 시간과 디바이스 시간 delta 계산, UI 표시에 보정 적용 | Firebase `serverTimestamp()`와 로컬 시간 비교로 delta 산출 |

### 3.3 "매시간" 정의

`spec/capture/overview.md:21-23` 확정 사항 기반:
- **정각 마감 방식** (사용자별 슬라이딩 아님)
- 09:00 슬롯 = 09:00:00 ~ 09:59:59 업로드 가능
- 10:00:00 도달 시 09:00 슬롯 마감, 소급 업로드 금지

**Firestore Security Rule 검증 로직:**
```javascript
// 의사코드
allow create: if
  request.auth.uid == resource.data.authorId
  && request.time >= slotStart
  && request.time < slotStart + 1hour + 30seconds  // grace
```

### 3.4 DST 처리

- 슬롯 키가 UTC 기반이면 DST 영향 없음
- 표시 계층에서만 DST 변환 적용 (`Intl.DateTimeFormat` 또는 `date-fns-tz`)
- **Spring Forward (시간 1시간 건너뜀)**: 로컬 02:00 슬롯이 사라짐 → UTC 기반이므로 실제로는 정상 존재, 표시만 건너뜀
- **Fall Back (시간 1시간 반복)**: 로컬 02:00이 2번 등장 → UTC 기반이므로 서로 다른 슬롯, 표시에 "(1차)/(2차)" 구분 또는 UTC 병기 필요

---

## 4. FCM 알림 폭주 방어

### 4.1 발송 전략

| 방식 | 적합도 | 설명 |
|---|---|---|
| FCM 토픽 (`/topics/group_{gid}`) | **권고** | 그룹 단위 구독. 서버에서 1회 발송 → FCM이 구독자에게 fan-out. 서버 부하 최소 |
| 개별 토큰 발송 (`sendMulticast`) | 비권고 | 5,000 DAU x 3 그룹 = 15,000 토큰 관리 + 배치 발송 500개 제한 → 30+ 배치 필요 |
| 조건부 토픽 | 향후 검토 | `'group_A' in topics && 'video_noti_on' in topics` 조합으로 알림 설정 반영 |

### 4.2 정각 thundering herd 문제

**문제**: 매시 정각에 1,800 활성 그룹에 동시 알림 발송 → Cloud Function cold start + FCM API 호출 집중.

**완화 전략:**

| # | 전략 | 설명 |
|---|---|---|
| 1 | **시간 분산 (jitter)** | 정각 이후 0~120초 랜덤 지연으로 발송 분산. 사용자 경험에 미미한 영향 |
| 2 | **Cloud Scheduler 분할** | 매시 정각 단일 트리거 대신, 00분/01분/02분에 그룹 ID 해시 기반 3분할 발송 |
| 3 | **Min instances** | Cloud Function 최소 인스턴스 2~3개 유지 (cold start 방지, 월 ~$5 추가) |
| 4 | **토픽 메시지** | 개별 발송 대비 서버 측 API 호출 1회/그룹으로 감소 |

### 4.3 iOS APNs 제약

| 항목 | 제약 | 대응 |
|---|---|---|
| 페이로드 한계 | 4,096 bytes | 3초 영상 메타만 포함 (URL 아님). 충분 |
| 사일런트 푸시 한계 | iOS가 **빈도 제한** 적용 (시간당 2~3회 권고) | 매시 정각 알림이 사일런트이면 1시간 1회로 제한 내. 단, 채팅 알림과 합산되면 초과 가능 → **영상 알림은 visible push로** |
| 백그라운드 앱 리프레시 | 사용자가 끌 수 있음 | 사일런트 푸시에 의존하지 말 것. 포그라운드 진입 시 fetch 보장 |
| Priority | `high` = 즉시, `normal` = 배터리 최적화 대상 | 영상 알림: `high`, 채팅: `high`, 다이제스트: `normal` |

### 4.4 알림 그룹화 / 다이제스트 / 조용한 시간

| 기능 | 구현 방안 |
|---|---|
| **그룹화** | Android: `notification.tag = "group_{gid}"` + `setGroup()`. iOS: `threadIdentifier = "group_{gid}"` |
| **다이제스트** | 사용자가 3개 그룹 소속 → 정각에 3건 알림 대신 1건 다이제스트 "3개 그룹에서 새 시간이 시작됐어요" 옵션. 서버에서 사용자별 그룹 수 확인 후 조건 분기 |
| **조용한 시간** | `spec/notifications/overview.md:14` 확정. `users/{uid}.quietHoursStart`, `quietHoursEnd` 필드. Cloud Function 발송 전 체크 |

---

## 5. Auth (전화번호) 비용/보안

### 5.1 SMS 비용

| 항목 | 값 |
|---|---|
| Firebase Phone Auth 무료 한도 | **월 10,000건** (2024년 기준, 정책 변동 확인 필요) |
| 초과 시 비용 | $0.01~0.06/건 (국가별 상이. 한국 ~$0.05) |
| 5,000 신규 인증/월 가정 | 무료 한도 내. 단, 재인증/기기 변경 포함 시 초과 가능 |
| 월 최악 시나리오 | 5,000 신규 + 2,000 재인증 + 3,000 재전송 = 10,000건 → 무료 한도 경계 |
| SMS 폭탄 공격 시 | 무제한 발송 → 월 수백 달러 피해 가능 |

### 5.2 보안 대응

| 위협 | 대응 |
|---|---|
| **SMS 폭탄 (toll fraud)** | Firebase App Check 필수 적용 (DeviceCheck/Play Integrity). reCAPTCHA Enterprise 연동. IP당/전화번호당 발송 제한 (Cloud Function rate limiter) |
| **SIM 스왑** | Firebase Phone Auth 자체로는 방어 불가. 고가치 계정에 2FA 추가(향후). 계정 이상 징후(새 디바이스 + 전화번호 변경) 시 세션 강제 만료 |
| **OTP 가로채기** | SSL Pinning (expo-secure-store + custom fetch). OTP 유효기간 단축 (기본 5분 → 가능하면 2분). Auto-fill로 사용자 직접 입력 최소화 |
| **전화번호 변경/이전** | Firebase는 전화번호 변경 시 새 UID 생성 위험 → `linkWithPhoneNumber`로 기존 계정에 새 번호 연결하는 흐름 필수 설계 |
| **reCAPTCHA** | iOS: APNs silent push가 기본 검증. reCAPTCHA는 fallback. Android: Play Integrity 기본 + reCAPTCHA fallback |

### 5.3 App Check 적용 권고

```
firebase.appCheck().activate(
  ReactNativeFirebase.ios: DeviceCheck,
  ReactNativeFirebase.android: PlayIntegrity
)
```
- Firestore/Storage/Functions 모두 App Check 토큰 요구로 설정
- 디버그 모드에서는 debug provider 사용

---

## 6. EAS 빌드/배포 리스크

### 6.1 Dev Client vs Prod 빌드 함정

| 항목 | Dev Client | Production | 함정 |
|---|---|---|---|
| JS 번들러 | Metro (개발 서버) | Hermes 바이트코드 | 성능 차이 10~50x. Dev에서 "느림"은 가짜 이슈 |
| 네이티브 모듈 | 동일 | 동일 | 단, ProGuard(Android) / Bitcode(iOS) 최적화로 Production에서 크래시 가능 |
| 환경변수 | `.env.development` | `.env.production` | Firebase 프로젝트 키 분리 필수 (개발/운영) |
| 디버깅 | Flipper/React DevTools 가능 | 불가 | Production 크래시는 Sentry/Crashlytics 필수 |
| APNs | Sandbox APNs | Production APNs | **Dev Client에서 푸시가 되어도 Prod에서 안 될 수 있음**. APNs 인증서/키 별도 확인 |

### 6.2 OTA 업데이트 한계

| 가능 | 불가능 |
|---|---|
| JS 코드 변경 | 네이티브 모듈 추가/변경 (`expo-camera` 버전 업 등) |
| 이미지/폰트 리소스 | `app.json` 설정 변경 (아이콘, 스플래시, 권한 문구) |
| React 컴포넌트 수정 | `@react-native-firebase` 네이티브 바인딩 변경 |

**권고**: 네이티브 의존성 변경 시 반드시 EAS Build 재실행. `eas update`와 `eas build` 호환성을 `runtimeVersion` 정책으로 관리 (`"runtimeVersion": { "policy": "fingerprint" }`).

### 6.3 앱스토어 심사 항목

| 항목 | 설명 | 대응 |
|---|---|---|
| **카메라/마이크 권한** | `NSCameraUsageDescription`, `NSMicrophoneUsageDescription` 필수. 구체적 사유 명시 ("매시간 3초 영상을 촬영하여 그룹에 공유합니다") | `app.json`의 `ios.infoPlist`에 명시 |
| **연락처 접근** | 그룹 초대 시 연락처 사용 → `NSContactsUsageDescription` | 사용 시점에만 요청 (lazy permission) |
| **백그라운드 모드** | `expo-task-manager` 사용 시 `UIBackgroundModes` 설정 | `fetch`, `remote-notification` 모드만 사용. `audio` 모드는 영상 재생이 포그라운드 전용이면 불필요 |
| **콘텐츠 정책** | 사용자 생성 콘텐츠(UGC) 앱 → 신고/차단 기능 필수 (Apple 가이드라인 1.2) | MVP에도 **신고/차단 최소 구현 필수**. 없으면 리젝 |
| **개인정보 라벨** | App Privacy 섹션에 전화번호, 사용자 콘텐츠(영상), 식별자 명시 | 정확한 라벨링 |
| **자동 재생** | 영상 자동 재생은 음소거 상태면 일반적으로 허용 | 현재 기획 "음소거 기본" 부합 |

---

## 7. 사이드 이팩트 리스트 (5,000명 기준 Critical Issues)

| # | 영역 | 사이드 이팩트 | 발생 조건 | 영향도 | 완화 전략 |
|---|---|---|---|---|---|
| 1 | Firestore | **정각 쓰기 폭주** — 매시 정각 후 5분 내 동시 업로드 → `groups/{gid}` 문서 `lastActivityAt` 갱신 충돌 | 5,000 DAU, 1,800 그룹이 정각 직후 동시 쓰기 | **상** | `lastActivityAt` 갱신을 Cloud Function 비동기 처리 + 1분 디바운스 |
| 2 | Firestore | **unreadVideoCount fan-out 폭발** — 새 영상당 7건 memberships 쓰기 × 1,800 그룹 = 12,600 writes/burst | 정각 직후 대량 업로드 시 | **상** | lastReadAt 기반 클라이언트 계산으로 전환 (쓰기 제거) |
| 3 | FCM | **정각 알림 thundering herd** — 1,800 그룹 × 토픽 메시지 = Cloud Function 동시 호출 급증 | 매시 정각 | **상** | jitter(0~120초) + Cloud Function min instances + 토픽 메시징 |
| 4 | Storage | **월 다운로드 비용 $300+** — 5,000 DAU가 영상 반복 조회 시 origin 전송량 급증 | 활발한 사용자 기반 | **중** | Firebase Hosting CDN 또는 Cloud CDN 연동, 썸네일 우선 로딩, lazy video load |
| 5 | Auth | **SMS 폭탄 (toll fraud)** — App Check 미적용 시 공격자가 SMS 발송 API 남용 | 앱 출시 후 봇 공격 | **상** | App Check + reCAPTCHA + IP/번호 rate limit 필수 |
| 6 | 시간동기화 | **슬롯 키 timezone 불일치** — 클라이언트 로컬 시간으로 슬롯 키 생성 시 그룹 내 다른 시간대 멤버와 슬롯 분리 | 다른 timezone 멤버가 같은 그룹 | **상** | UTC 기반 슬롯 키 저장 + 표시만 로컬 변환 |
| 7 | Storage | **7일 삭제 실패 시 비용 누적** — Lifecycle rule 또는 Function 실패 시 영상 무한 누적 | Cloud Function 오류, lifecycle 미적용 | **중** | 이중 삭제 (Storage lifecycle + Function), 모니터링 알림 |
| 8 | 채팅 | **Firestore 리스너 비용** — 5,000 DAU × 3 그룹 × 채팅 리스너 = 15,000 동시 리스너 → 읽기 비용 급증 | 채팅 활성화 시 | **중** | 채팅 화면 진입 시에만 리스너 연결, 퇴장 시 즉시 해제. 페이지네이션 50건 |
| 9 | EAS | **OTA 업데이트 후 네이티브 불일치** — JS 업데이트가 네이티브 변경을 요구하는 경우 크래시 | 네이티브 의존성 변경 + OTA만 배포 | **중** | `runtimeVersion: fingerprint` 정책으로 자동 감지 |
| 10 | UGC | **앱스토어 리젝 — 신고/차단 미구현** — Apple 가이드라인 1.2 위반 | MVP 출시 시 | **상** | 최소 신고 버튼 + 관리자 검토 큐 구현 필수 |

---

## 8. 기획자/테스터에게 요청 사항

### 8.1 기획자에게 결정 요청

| # | 결정 항목 | 배경 | 영향 범위 |
|---|---|---|---|
| 1 | **슬롯 키의 시간대 기준** — UTC 저장 + 로컬 표시 방식 확정 여부 | `spec/group-detail/data.md:24-26`에 "사용자 로컬" 명시, 그러나 다른 시간대 멤버 공존 시 버그 | 데이터 모델, 보안 룰, 전체 시간 표시 |
| 2 | **한 사용자의 최대 그룹 수** — 미정(`decisions.md:73`). 비용/성능에 직접 영향 | 그룹 수 무제한 시 1인당 Firestore 읽기/FCM 토픽 구독 무한 증가 | Firestore 비용, FCM 구독, 메인 화면 성능 |
| 3 | **UGC 신고/차단 정책** — MVP에 필수이나 기획 미존재 | Apple 가이드라인 1.2 미준수 시 리젝 확정 | 전체 화면 (영상 + 채팅), 관리 도구 |
| 4 | **채팅 페이지네이션 단위 및 메시지 삭제 시간 제한** — `spec/chat/overview.md:27-28` 미정 | 페이지네이션 단위는 Firestore 읽기 비용에 직접 영향, 삭제 시간 제한은 보안 룰 설계에 영향 | 채팅 데이터 모델, 보안 룰 |
| 5 | **푸시 알림 정책 세부** — 정각 트리거 시점, 다이제스트 여부, 그룹별 설정 | `decisions.md:71` 미정. FCM 설계와 Cloud Function 스케줄링에 선행 필요 | FCM, Cloud Functions, notifications 화면 |

### 8.2 테스터에게 검증 요청

| # | 검증 항목 | 검증 방법 | 판정 기준 |
|---|---|---|---|
| 1 | **정각 동시 업로드 부하** | 50+ 동시 클라이언트로 정각 직후 3초 영상 동시 업로드 시뮬레이션 | Firestore 쓰기 실패율 < 1%, 평균 업로드 완료 시간 < 10초 |
| 2 | **슬롯 마감 경계 테스트** | 정각 직전 (-1초, -0.5초) 및 직후 (+0.5초, +1초, +30초) 업로드 시도 | grace 윈도우 내 허용, 초과 시 정확히 거부 |
| 3 | **오프라인 → 온라인 업로드 큐 복원** | 녹화 후 비행기 모드 → 앱 종료 → 재시작 → 네트워크 복원 → 자동 업로드 확인 | 큐 항목 유실 없이 자동 업로드 완료, 슬롯 마감 후에는 정상 제거 |
| 4 | **다중 시간대 그룹 영상 표시** | 서울(UTC+9)과 도쿄(UTC+9)/LA(UTC-7) 사용자가 같은 그룹에서 영상 업로드 및 조회 | 같은 슬롯에 영상이 묶이고, 각 사용자에게 로컬 시간으로 정확히 표시 |
| 5 | **FCM 알림 수신율** | 5,000 디바이스 (에뮬레이터 + 실기기 혼합)에 토픽 메시지 발송 후 수신 확인 | 수신율 > 95% (iOS 저전력 모드 제외), 지연 < 5초 |

---

## Critical Issue Top 5 요약

1. **[상] 시간대 기준 미확정** — `spec/group-detail/data.md:24-26`의 "사용자 로컬" 방식이 그대로 구현되면 다른 시간대 멤버 간 슬롯 분리 버그 발생. **UTC 저장 + 로컬 표시 확정 필수**.
2. **[상] 정각 Firestore 쓰기 폭주** — `groups/{gid}.lastActivityAt` + `memberships.unreadVideoCount` fan-out이 Firestore 1초 1문서 1회 쓰기 제한에 충돌. **lastReadAt 기반 클라이언트 계산으로 전환 필요**.
3. **[상] SMS toll fraud (App Check 미적용)** — Firebase Phone Auth에 App Check 없이 출시하면 SMS 비용 폭탄 위험. **Day-1부터 App Check + rate limit 필수**.
4. **[상] 앱스토어 리젝 위험 (UGC 신고/차단 미구현)** — Apple 가이드라인 1.2에 의해 사용자 생성 콘텐츠 앱은 신고/차단 기능 필수. 현재 기획서에 미존재. **MVP에 최소 구현 필수**.
5. **[중] Storage 다운로드 비용 $300+/월** — CDN 캐시 없이 5,000 DAU의 영상 조회 시 월 $324 추정. **CDN + 썸네일 우선 로딩 + lazy video load로 60~70% 절감 가능**.

---

## References

- `spec/main/data.md:7-9` -- Firestore groups 쿼리 구조
- `spec/main/data.md:16` -- memberships unreadVideoCount 필드
- `spec/main/data.md:35` -- 복합 인덱스 명세
- `spec/group-detail/data.md:14-18` -- posts 서브컬렉션 쿼리
- `spec/group-detail/data.md:24-26` -- hourSlot 타임존 "사용자 로컬" 명시
- `spec/group-detail/data.md:50-51` -- 인덱스 설계
- `spec/capture/overview.md:21-23` -- 업로드 윈도우 정각 정책
- `spec/capture/overview.md:27-29` -- 업로드 큐 정책
- `spec/chat/overview.md:22` -- "모두 읽음" 정책
- `spec/chat/overview.md:27-28` -- 페이지네이션/삭제 미정
- `spec/notifications/overview.md:14` -- 방해 금지 시간
- `docs/2026-05-11/decisions.md:49-51` -- 그룹 8명, 60분 윈도우, 7일 보존
- `docs/2026-05-11/decisions.md:71-74` -- 미정 사항 (초대, 푸시, 그룹 수 상한)
