# infra/ — 인프라 설정 파일

ChimeMe 배포에 필요한 Firebase / EAS / Expo 설정 파일 모음.

## 파일 목록

| 파일 | 용도 | 배포 명령 |
|---|---|---|
| `firestore.rules` | Firestore Security Rules | `firebase deploy --only firestore:rules` |
| `firestore.indexes.json` | Firestore 복합 인덱스 | `firebase deploy --only firestore:indexes` |
| `storage.rules` | Cloud Storage Security Rules | `firebase deploy --only storage:rules` |
| `firebase.json` | Firebase CLI 통합 설정 | (CLI가 자동 참조) |
| `eas.json` | EAS Build 프로파일 | (EAS CLI가 자동 참조) |
| `app.json.template` | Expo 앱 설정 템플릿 | 프로젝트 루트로 복사 후 값 채우기 |
| `.env.example` | 환경변수 템플릿 | `.env.development` / `.env.production`로 복사 |

## 첫 배포 시 순서

```bash
# 1. Firebase CLI 로그인
firebase login

# 2. 프로젝트 선택
firebase use chimeme-dev   # 개발 환경
# 또는
firebase use chimeme-prod  # 프로덕션

# 3. Rules + Indexes 배포
firebase deploy --only firestore:rules,firestore:indexes,storage:rules

# 4. (Phase 4) Cloud Functions 배포
firebase deploy --only functions

# 5. (Phase 4) Hosting 배포 (약관 페이지)
firebase deploy --only hosting
```

## 환경 변수 설정

```bash
# 1. .env.example를 복사
cp infra/.env.example .env.development
cp infra/.env.example .env.production

# 2. 각 .env 파일에 Firebase 콘솔에서 받은 값 입력
# 3. EAS Secrets에 등록 (빌드 시 주입용)
eas secret:create --scope project --name FIREBASE_API_KEY_DEV --value "AIza..."
eas secret:create --scope project --name FIREBASE_API_KEY_PROD --value "AIza..."
# ... 다른 변수도 반복
```

## Security Rules 변경 시 검증

```bash
# 로컬 Emulator에서 테스트
firebase emulators:start --only firestore,storage

# 별도 터미널에서 Rules Test Suite 실행
npm test -- src/__tests__/rules/
```

## 인덱스 생성 시간

- 작은 컬렉션 (< 1000 docs): 즉시
- 큰 컬렉션 (10,000+ docs): 수 분~수 시간
- 배포 후 Firebase 콘솔 → Firestore → 인덱스에서 상태 확인

## 보안 주의사항

- 본 디렉토리의 파일은 git에 커밋됨 (오픈)
- 실제 값(`.env.*`, `GoogleService-Info.plist`, `google-services.json`)은 git 제외
- `infra/google-play-service-account.json`은 EAS Secrets로 관리

## App Check 활성화 후 주의

`firestore.rules` / `storage.rules`는 App Check 무관하게 동작.
App Check enforcement는 Firebase Console에서 별도 설정.

## 변경 이력

- 2026-05-12 — Phase 3 초기 작성 (D-12-01~13 반영)
