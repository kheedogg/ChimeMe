# 2026-05-11 — 진행 기록

## 오늘 한 일

### 의사결정 확정
- 앱 이름: **ChimeMe** (Chime은 동명 핀테크 서비스 충돌로 폐기)
- 백엔드: Firebase (Auth · Firestore · Storage · FCM)
- 인증: 전화번호 단일 방식 (Firebase Phone Auth)
- 채팅 범위: **그룹 채팅만** (DM 없음)
- 클라이언트: **React Native + Expo + EAS Dev Client**
- Firebase 클라이언트 SDK: **@react-native-firebase** (네이티브 Phone Auth)
- Bundle ID / Package: `com.chimeme.mobile`
- 디렉토리 구조: 루트 평면 배치 (서브폴더 없이 `app/`, `src/` 등을 루트에)

### 작업 규칙 확립
- **CLAUDE.md** 갱신 — 프로젝트 개요 + 디렉토리 규칙 + 디자인 토큰
- Skill 등록 — `.claude/skills/chimeme-screen-spec/SKILL.md`
  - 앱 기획서는 `spec/<screen-key>/`에 화면별 폴더로
  - 실행 기록은 `docs/YYYY-MM-DD/`에 날짜별로
- `spec/README.md` 인덱스 작성

### 기획서 작성 (`spec/`)
- **main/** — 4개 표준 문서 1차 완성 (mainPage.png 기반)
  - overview.md, ui.md, interactions.md, data.md
- **group-detail/** — 4개 표준 문서 1차 완성 (groupPage.png / groupPage2.png 기반)
  - overview.md, ui.md, interactions.md, data.md
- 스텁(overview.md만):
  - group-create, auth-phone, auth-verify, capture, chat, profile, notifications

### docs/2026-05-11/
- plan.md, naming.md, decisions.md (기존)
- progress.md (이 문서)

## 다음 작업 (다음 세션 우선순위)

1. **Expo 프로젝트 스캐폴딩** (오늘 미완)
   - `npx create-expo-app` + expo-router 셋업
   - 의존성 설치 (@react-native-firebase, expo-camera 등)
   - `app.json` / `eas.json` 작성, Bundle ID `com.chimeme.mobile` 등록
   - 최소 boilerplate (auth/main 라우팅 + GroupCard 컴포넌트)
2. **Firebase Console 세팅** (사용자 수동 + 가이드)
   - ChimeMe 프로젝트 생성, iOS/Android 앱 등록, Phone Auth 활성화
   - asia-northeast3 (서울) 리전 Firestore + Storage
3. 미작성 스텁 화면의 ui/interactions/data.md 채우기 (구현 직전에)
4. 도메인 / 앱스토어 / 상표 사전 조회 (chimeme.app, chimeme.io 등)

## 미해결 이슈
- 그룹 최대 인원
- 슬롯 마감 시간 (정각 + N분?)
- 영상 보존 기간
- 베타 SMS 무료 한도 운영 정책
