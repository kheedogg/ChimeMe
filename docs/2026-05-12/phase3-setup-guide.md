# Phase 3 — 인프라 / 계정 / 법적 문서 셋업 가이드

> 사용자가 직접 수행해야 하는 셋업을 단계별로 안내한다. 각 단계의 ✓는 완료 체크용. ⚠️은 비용/주의 사항.

---

## 0. 사전 준비

| 항목 | 비용 | 필요 시간 |
|---|---|---|
| Google 계정 (Firebase / Play용) | 무료 | — |
| Apple ID + 결제 카드 (Apple Developer) | $99/년 | 등록 24~48시간 |
| 결제 카드 (Firebase Blaze) | 사용량 기반 (~$5/월 시작) | 즉시 |
| 도메인 (`chimeme.app` 등) | $10~30/년 | 즉시 |

---

## 1. Firebase 프로젝트 셋업 (dev + prod 2개)

### 1.1 프로젝트 생성

**경로**: https://console.firebase.google.com → "프로젝트 추가"

- [ ] `chimeme-dev` 프로젝트 생성
- [ ] `chimeme-prod` 프로젝트 생성
- [ ] 각 프로젝트의 **프로젝트 ID** 메모 (코드에서 사용)

### 1.2 Blaze 플랜 업그레이드

> ⚠️ Cloud Functions, App Check, 외부 API 호출에 필수.

- [ ] 좌측 하단 "Spark 무료 플랜" → **Blaze 플랜으로 업그레이드**
- [ ] 결제 카드 등록
- [ ] **예산 알림 설정**: 좌상단 ⚙️ → 사용량 및 청구 → 예산 알림 설정
  - 알림 1: $50 (50% 도달)
  - 알림 2: $100 (정상 사용량 상한)
  - 알림 3: $200 (이상 사용량 감지)

### 1.3 Firestore Database 활성화

- [ ] 좌측 메뉴 "Firestore Database" → "데이터베이스 만들기"
- [ ] **위치 선택**: `asia-northeast3 (서울)` ⚠️ **이후 변경 불가**
- [ ] 시작 모드: **프로덕션 모드** (Security Rules 직접 작성)
- [ ] dev/prod 모두 동일하게 설정

### 1.4 Storage 활성화

- [ ] 좌측 메뉴 "Storage" → "시작하기"
- [ ] **위치**: Firestore와 동일 리전 (`asia-northeast3 (서울)`)
- [ ] 시작 모드: 프로덕션 모드

### 1.5 Authentication > Phone 활성화

- [ ] 좌측 메뉴 "Authentication" → "시작하기"
- [ ] 로그인 방법 탭 → **전화** 활성화
- [ ] (선택) 테스트 전화번호 등록 (개발용)

### 1.6 Cloud Messaging (FCM) 활성화

- [ ] "Cloud Messaging" 메뉴 → 활성화
- [ ] **iOS 푸시 인증 키 업로드** (Apple Developer에서 발급 후 — §4 참조)
- [ ] Android는 자동 활성화

### 1.7 Cloud Functions 활성화

- [ ] "Functions" 메뉴 → "Functions 사용 설정"
- [ ] Node.js 18+ 또는 Python 3 환경 (Node 권장)
- [ ] 리전: `asia-northeast3`

### 1.8 App Check 활성화 (D-12-04 필수)

- [ ] "App Check" 메뉴 → 시작하기
- [ ] iOS 앱 등록 → 공급업체 **DeviceCheck** 선택
- [ ] Android 앱 등록 → 공급업체 **Play Integrity** 선택
- [ ] 서비스별 적용 토글 ON:
  - Cloud Firestore
  - Cloud Storage for Firebase
  - Cloud Functions
  - Authentication
- [ ] **enforcement 모드**: 베타는 "보고 전용" → 출시 시 "적용" 전환

### 1.9 Firebase Hosting (약관 페이지용)

- [ ] "Hosting" 메뉴 → 시작하기
- [ ] 사용자 정의 도메인: `chimeme.app` (DNS 설정 필요)
- [ ] 약관 파일을 `public/legal/tos/v1.html` 등으로 배포

### 1.10 보안 규칙 / 인덱스 배포

> 다음 단계는 로컬에서 Firebase CLI 사용 (§7 참조).
> 본 가이드 첨부 파일 사용:
> - `infra/firestore.rules` — 통합 Security Rules
> - `infra/storage.rules` — Storage Rules
> - `infra/firestore.indexes.json` — 복합 인덱스

```bash
firebase use chimeme-dev
firebase deploy --only firestore:rules,firestore:indexes,storage:rules
```

---

## 2. Apple Developer Program 등록

### 2.1 계정 가입

**경로**: https://developer.apple.com/programs/enroll/

- [ ] Apple ID로 로그인
- [ ] Individual / Organization 선택
- [ ] $99/년 결제
- [ ] 등록 승인 대기 (24~48시간)

### 2.2 App ID 생성

**경로**: Developer Account → Certificates, IDs & Profiles → Identifiers

- [ ] "+" 클릭 → App IDs → App
- [ ] Bundle ID: **`com.chimeme.mobile`** (Explicit)
- [ ] Capabilities 체크:
  - Push Notifications
  - Sign in with Apple (선택)
  - Associated Domains (Universal Links — `applinks:chimeme.app`)

### 2.3 APNs Authentication Key 생성

**경로**: Certificates, IDs & Profiles → Keys

- [ ] "+" 클릭 → Apple Push Notifications service (APNs) 체크
- [ ] Key 이름: `ChimeMe APNs`
- [ ] **`.p8` 파일 다운로드** (⚠️ 1회만 가능 — 안전한 곳에 백업)
- [ ] **Key ID** 메모 (Firebase에 업로드 시 필요)
- [ ] **Team ID** 메모 (Apple Developer 페이지 우상단)

### 2.4 Firebase에 APNs 업로드

- [ ] Firebase 콘솔 → Cloud Messaging → iOS 앱 → APNs 인증 키 업로드
- [ ] `.p8` 파일 + Key ID + Team ID 입력

### 2.5 Universal Links (그룹 초대 딥링크)

**경로**: Developer Account → App IDs → 본 앱 편집 → Capabilities

- [ ] Associated Domains 추가: `applinks:chimeme.app`
- [ ] 웹 측에서 `https://chimeme.app/.well-known/apple-app-site-association` 호스팅:
  ```json
  {
    "applinks": {
      "apps": [],
      "details": [{
        "appID": "TEAM_ID.com.chimeme.mobile",
        "paths": ["/invite/*"]
      }]
    }
  }
  ```

---

## 3. Google Play Console 등록

### 3.1 개발자 계정 등록

**경로**: https://play.google.com/console/signup

- [ ] Google 계정으로 로그인
- [ ] **$25 1회 결제**
- [ ] 개인/기관 선택 → 검증 절차 (신분증 등 필요할 수 있음)
- [ ] 등록 승인 대기 (1~3일)

### 3.2 앱 등록

**경로**: Play Console → 앱 만들기

- [ ] 앱 이름: `ChimeMe`
- [ ] 기본 언어: 한국어
- [ ] 앱 또는 게임: 앱
- [ ] 무료/유료: 무료

### 3.3 패키지 이름 / 서명 키

- [ ] 패키지 이름: **`com.chimeme.mobile`** (Bundle ID와 동일)
- [ ] **앱 서명 키**: Play App Signing 사용 (Google이 관리)
- [ ] EAS Build로 빌드 시 keystore 자동 생성

### 3.4 SHA-1 / SHA-256 등록 (Firebase 연동)

- [ ] EAS 빌드 후 `eas credentials` 명령으로 SHA-1/256 추출:
  ```bash
  eas credentials -p android
  ```
- [ ] Firebase 콘솔 → 프로젝트 설정 → Android 앱 → 디지털 지문 추가

### 3.5 App Links (그룹 초대 딥링크)

- [ ] Play Console → 앱 정보 → 앱 콘텐츠 → 딥 링크
- [ ] 웹 측에서 `https://chimeme.app/.well-known/assetlinks.json` 호스팅:
  ```json
  [{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.chimeme.mobile",
      "sha256_cert_fingerprints": ["AA:BB:CC:..."]
    }
  }]
  ```

---

## 4. EAS (Expo Application Services) 셋업

### 4.1 Expo 계정 + 로그인

```bash
npm install -g eas-cli
eas login
```

### 4.2 프로젝트 초기화 (Phase 4에서 수행 예정)

```bash
cd /Users/doheekim/home/MyApp
npx create-expo-app . --template
eas init
```

### 4.3 환경변수 / 시크릿 등록

- [ ] `infra/.env.example` 참고하여 dev/prod 분리
- [ ] EAS Secrets 등록:
  ```bash
  eas secret:create --scope project --name FIREBASE_API_KEY_DEV --value xxxxx
  eas secret:create --scope project --name FIREBASE_API_KEY_PROD --value yyyyy
  # ... 다른 시크릿도 반복
  ```

### 4.4 빌드 프로파일 (`eas.json`)

- [ ] 본 가이드 첨부 `infra/eas.json` 사용
- [ ] dev / preview / production 3개 프로파일

### 4.5 Development Client 빌드

```bash
eas build --profile development --platform ios
eas build --profile development --platform android
```

> ⚠️ 첫 빌드는 30분~1시간 소요. 무료 할당량(월 30회) 초과 시 유료.

---

## 5. 이용약관 / 개인정보처리방침 작성

### 5.1 초안 위치

- [ ] `legal-drafts/tos-v1.md` — 이용약관 초안
- [ ] `legal-drafts/privacy-v1.md` — 개인정보처리방침 초안

> ⚠️ **법무 검토 필수**. 한국 출시는 KISA / 방통위 가이드라인 준수.

### 5.2 검토 체크리스트

- [ ] 회사명 / 사업자 정보 채우기 (`[CHIMEME_ENTITY_NAME]` placeholder)
- [ ] 대표자 이름 / 주소 / 연락처
- [ ] 사업자등록번호 (해당 시)
- [ ] 개인정보보호 책임자 지정
- [ ] 변호사 / 법무팀 검토
- [ ] 약관 변경 시 사전 공지 절차 정의

### 5.3 호스팅 (Firebase Hosting)

```bash
# legal-drafts/*.md → HTML 변환 후 public/legal/에 배치
firebase deploy --only hosting
```

배포 후 URL:
- 이용약관: `https://chimeme.app/legal/tos/v1.html`
- 개인정보처리방침: `https://chimeme.app/legal/privacy/v1.html`

### 5.4 Remote Config에 버전 등록

```json
{
  "legal": {
    "tos": {
      "currentVersion": "v1",
      "effectiveDate": "2026-05-12",
      "url": "https://chimeme.app/legal/tos/v1.html"
    },
    "privacy": {
      "currentVersion": "v1",
      "effectiveDate": "2026-05-12",
      "url": "https://chimeme.app/legal/privacy/v1.html"
    }
  }
}
```

---

## 6. 도메인 / DNS

### 6.1 도메인 구매

- [ ] `chimeme.app` (.app TLD 권장 — 자동 HTTPS) 또는 대체 도메인
- [ ] Gabia / Cloudflare / Google Domains 등에서 구매

### 6.2 Firebase Hosting 연결

- [ ] Firebase 콘솔 → Hosting → 사용자 정의 도메인 추가
- [ ] TXT 레코드 추가 (소유권 확인)
- [ ] A 레코드 또는 CNAME 추가 (Firebase 제공 IP)
- [ ] HTTPS 자동 발급 대기 (~1시간)

### 6.3 .well-known 파일 호스팅
- [ ] `public/.well-known/apple-app-site-association` (iOS Universal Links)
- [ ] `public/.well-known/assetlinks.json` (Android App Links)

---

## 7. 로컬 개발 환경

### 7.1 필수 도구

```bash
# Node.js 18+ (LTS)
nvm install --lts

# Firebase CLI
npm install -g firebase-tools
firebase login

# EAS CLI
npm install -g eas-cli
eas login

# Java 17 (Android 빌드용, Mac은 Homebrew)
brew install --cask temurin@17
```

### 7.2 Firebase 프로젝트 초기화

```bash
cd /Users/doheekim/home/MyApp
firebase init
# 선택: Firestore, Functions, Hosting, Storage, Emulators
# 프로젝트: chimeme-dev (default)
```

### 7.3 Emulator Suite 실행 (개발/테스트용)

```bash
firebase emulators:start --only auth,firestore,storage,functions,pubsub
```

---

## 8. 검증 체크리스트 (Phase 3 완료 판정)

### Firebase
- [ ] dev / prod 두 프로젝트 생성 완료
- [ ] Blaze 플랜 활성화 + 예산 알림 3종 설정
- [ ] Firestore / Storage / FCM / Functions 활성화
- [ ] App Check 등록 (DeviceCheck + Play Integrity)
- [ ] Security Rules + Indexes 배포

### 스토어
- [ ] Apple Developer Program 가입 완료
- [ ] APNs `.p8` Key 발급 + 백업
- [ ] Firebase에 APNs Key 업로드
- [ ] Google Play Console 가입 완료
- [ ] App IDs / Package Name 등록

### EAS
- [ ] Expo 계정 + EAS Login
- [ ] EAS Secrets 등록 (dev/prod 분리)
- [ ] `eas.json` 빌드 프로파일 작성

### 법적 문서
- [ ] 이용약관 v1 작성 + 법무 검토
- [ ] 개인정보처리방침 v1 작성 + 법무 검토
- [ ] Firebase Hosting에 배포
- [ ] Remote Config에 버전 등록

### 도메인
- [ ] 도메인 구매 + DNS 설정
- [ ] Firebase Hosting 연결 + HTTPS 인증서 발급
- [ ] `.well-known` 파일 호스팅 (Universal Links + App Links)

### 보안 키 백업
- [ ] APNs `.p8` 파일 — 2곳 이상 백업
- [ ] Android signing key (EAS 관리 시 자동, self-managed 시 수동 백업)
- [ ] Firebase Admin SDK service account JSON
- [ ] Firebase API Keys (dev/prod 분리)

---

## 9. 예상 비용 (월간)

| 항목 | 베타 (~100 사용자) | 5,000 DAU |
|---|---|---|
| Firebase Firestore | $0 (무료 한도) | ~$30 |
| Firebase Storage | $0 (10GB 미만) | ~$330 |
| Firebase Functions | $0 (무료 한도) | ~$20 |
| Firebase Auth (SMS) | $0 (10K 미만) | ~$50 |
| Firebase Hosting | $0 | ~$5 |
| App Check | $0 | $0 |
| EAS Build | $0 (월 30회) | $19 (Production 플랜) |
| Apple Developer | $8.25/월 (연 $99) | $8.25 |
| Google Play | 무시 (1회 $25) | — |
| 도메인 | ~$1.5/월 | ~$1.5 |
| **합계** | **~$10** | **~$464** |

> 비용 절감 포인트:
> - Storage 다운로드 비용이 압도적 → CDN 캐시 적용 시 60~70% 절감
> - EAS Build는 월 30회 무료, Phase 4 초기에는 충분

---

## 10. 다음 단계 (Phase 4)

Phase 3 완료 후 진행:
- [ ] Expo 프로젝트 스캐폴딩
- [ ] Firebase SDK 통합
- [ ] Firestore Security Rules 실 배포 (Emulator 테스트 후)
- [ ] Cloud Functions 구현 (spec/*/data.md 참조)
- [ ] 첫 EAS Dev Client 빌드 + 실기기 테스트
- [ ] Firebase Emulator + k6 부하 테스트 환경 구성

---

## 11. 자주 발생하는 문제 / 트러블슈팅

### Firebase 콘솔
- **Q. Firestore 리전을 잘못 선택했어요**
  - A. 프로젝트 자체를 새로 만들어야 합니다. 데이터 없는 초기 단계라 손실 없음.

- **Q. App Check에서 디버그 빌드가 차단돼요**
  - A. 디버그 빌드는 debug provider 사용. enforcement 모드를 "보고 전용"으로 시작.

### EAS Build
- **Q. iOS 빌드가 인증서 오류로 실패해요**
  - A. `eas credentials` 명령으로 인증서 재발급. 또는 Apple Developer에서 만료 확인.

### Apple Developer
- **Q. 등록 승인이 늦어요 (48시간 초과)**
  - A. Apple Support에 문의. 보통 7일 이내 해결.

### 약관 / 법무
- **Q. 약관을 어디까지 직접 작성해도 되나요?**
  - A. 초안은 본 가이드 첨부 사용. 출시 전 반드시 법무 검토.
