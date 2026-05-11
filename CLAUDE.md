# MyApp — ChimeMe 프로젝트

## 프로젝트 개요

**ChimeMe** — SetLog 컨셉을 모티브로 한 그룹 일상 공유 앱. (기존 후보 "Chime"은 동명 핀테크 서비스와 충돌 → ChimeMe로 확정.)
- 친구·가족 그룹을 만들어 **매 시간마다 3초 영상**을 공유
- **그룹 채팅** 지원 (DM 없음)
- 전화번호 인증 단일 방식
- iOS + Android 동시 빠른 베타 배포 목표 (React Native + Expo + EAS Dev Client)
- 백엔드: Firebase (Auth · Firestore · Storage · FCM)
- Bundle ID / Package: `com.chimeme.mobile`

## 디렉토리 규칙 (필수 준수)

### 1. 작업 실행 기록 — `docs/YYYY-MM-DD/`
모든 계획 · 진행 · 의사결정은 날짜별 폴더에 기록한다.
- 형식: `docs/2026-05-11/`, `docs/2026-05-12/` …
- 표준 파일: `plan.md`, `progress.md`, `decisions.md`, (선택) `naming.md`, `<screen-key>.md`
- 하루 단위로 그날의 작업 흐름을 추적할 수 있어야 함

### 2. 앱 기획서 — `spec/<screen-key>/`
**전체적인 앱 기획서는 화면별로 따로 폴더를 구성하여 기록한다.**
- 모든 화면 기획은 `spec/<screen-key>/` 폴더 하위에 둔다
- 화면별 표준 구성:
  - `overview.md` — 목적 · 사용자 시나리오
  - `ui.md` — UI 구성요소 · 레이아웃 · 디자인 토큰
  - `interactions.md` — 인터랙션 · 상태 전이
  - `data.md` — 사용/생성 데이터 · 쿼리
- 인덱스는 `spec/README.md`에 유지

### 3. 화면 키 명명 규칙
- kebab-case, 영문, 기능 기반
- 현재 정의: `main`, `group-detail`, `group-create`, `auth-phone`, `auth-verify`, `onboarding`, `capture`, `chat`, `profile`, `notifications`

### 4. 기획서와 실행 기록의 분리
- **기획서(spec/)** : 화면이 "무엇이어야 하는지" — 변경되면 spec 직접 수정
- **실행 기록(docs/)** : 그날 "무엇을 했는지" — 추가만, 수정/삭제 지양

## 디자인 시스템 토큰 (확정)
- 메인 컬러: 따뜻한 노란색 `#FFD54F` 톤
- 보조: 화이트 + 옅은 그레이 + 짙은 헤딩
- 카드: 라운드 + 옅은 보더 + 미세 그림자
- 헤딩: 굵은 sans-serif

## 자주 쓰는 명령어 (스캐폴딩 후 추가)
- `npm start` — Metro 번들러 시작
- `eas build --profile development --platform ios` — iOS Dev Client 빌드
- `eas update --branch preview` — OTA 업데이트

## 관련 Skill
- `chimeme-screen-spec` — 화면 기획 + 실행 기록 워크플로 (`.claude/skills/chimeme-screen-spec/`)
