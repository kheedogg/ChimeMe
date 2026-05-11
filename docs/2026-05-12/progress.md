# 2026-05-12 진행 기록

> 오늘 실제로 수행한 작업을 시간 순으로 기록한다. 결정은 `decisions.md`, 계획은 `plan.md` 참조.

---

## 09:00~ 3인 에이전트 팀 구성 및 병렬 리뷰

### 팀 구성
- **기획자** (planner@opus) — Product Planner 역할
- **개발자** (architect@opus) — Senior Developer / 아키텍처 검토
- **테스터** (test-engineer@sonnet) — QA Engineer / 엣지케이스 발굴

### 산출물
- `docs/2026-05-12/review-planner.md` — 화면 완성도 매트릭스, 비즈니스 갭 9영역, 우선순위 P0~P2
- `docs/2026-05-12/review-developer.md` — Firestore/Storage/FCM/Auth/EAS 분석, Critical Issue Top 5
- `docs/2026-05-12/review-tester.md` — 엣지케이스 카탈로그 A~J(60+ 시나리오), 부하 시나리오 4, P0~P3 매트릭스

> 모든 에이전트 Write 권한이 거부되어 콘텐츠를 텍스트로 반환받아 메인 에이전트가 직접 파일 작성.

---

## 13:00~ 깃 저장소 구성

### 문제 발견
- 기존 git이 `/Users/doheekim/` (홈 디렉토리)에 있어 민감 파일 노출 위험
- 원격이 `doheeyaa/git-test.git`으로 요청 원격(`kheedogg/ChimeMe.git`)과 다름

### 옵션 A 채택 (MyApp/ 별도 저장소)
1. `git init` (main 브랜치)
2. `.gitignore` 작성 (`.DS_Store`, `node_modules/`, `.expo/`, `.omc/`, `.env*`, Firebase config 등)
3. 안전 디렉토리만 명시적 스테이지 (`spec/`, `docs/`, `.claude/skills/`, `CLAUDE.md`, `AppPlanningDoc/`)
4. `git remote add origin https://github.com/kheedogg/ChimeMe.git`

### Push 시도 흐름
- 첫 시도: 원격에 GitHub 자동 생성 README가 있어 fast-forward rejected
- `git pull --rebase --allow-unrelated-histories origin main` → README 보존 + 로컬 commit 위로 rebase
- `gh auth login` 인증 (PAT 노출 이슈 발생 → revoke 권고)
- 최종 push 성공

---

## 14:00~ 3인 리뷰 통합 → plan.md + decisions.md

### plan.md 작성
- 3인 합의된 P0 5개 (C1~C5) 도출
- 단계별 실행 계획 Phase 0~4
- 리스크 / 블로커 정리

### decisions.md 작성 → 확정
- §A: 3인 합의 즉시 적용 12개 (D-12-01~12)
- §B: 사용자 결정 대기 12개 (Q-12-01~12)
- §C: 2026-05-11 미정 사항 4개 해소 매핑
- §D: 결정 영향으로 수정 필요한 spec 파일 매트릭스
- §E: 미해결 / 추후 검토

### 사용자 결정 12개 답변
- 모두 추천안 채택 → Q-12-01~12 전부 확정 처리
- 두 번째 푸시 (`docs/2026-05-12/decisions.md` 갱신)

---

## 17:00~ Phase 1 — 스텁 8개 화면 보강

### 신규 작성 (24개 파일)
| 화면 | ui.md | interactions.md | data.md |
|---|---|---|---|
| auth-phone | ✅ | ✅ | ✅ |
| auth-verify | ✅ | ✅ | ✅ |
| onboarding | ✅ | ✅ | ✅ |
| group-create | ✅ | ✅ | ✅ |
| capture | ✅ | ✅ | ✅ |
| chat | ✅ | ✅ | ✅ |
| profile | ✅ | ✅ | ✅ |
| notifications | ✅ | ✅ | ✅ |

### decisions.md §D 적용 (기존 파일 업데이트)
- `spec/group-detail/data.md` — `hourSlot` UTC 저장 + 표시만 로컬 (D-12-01) / grace 30초 (D-12-02)
- `spec/main/data.md` — `unreadVideoCount` → `lastReadVideoAt` 클라이언트 계산 (D-12-05)
- `spec/main/interactions.md` — 정렬/판정 클라이언트 측 전환 (D-12-05)
- `spec/capture/overview.md` — 720p H.264 인코딩 표준 + 큐 정책 확정 (D-12-03, D-12-07)
- `spec/chat/overview.md` — 50건 페이지네이션 + 5분 삭제 제한 + 90일 보존 + 신고 정책 (D-12-08, Q-12-04/05/08)
- `spec/notifications/overview.md` — 기본값 / FCM 발송 체크 순서 (Q-12-09, D-12-06)

### 인덱스 갱신
- `spec/README.md` — 10/10 화면 1차 완성으로 상태 갱신, 핵심 정책 요약 추가

### 최종 깃 푸시
- 신규/수정 파일 일괄 커밋 → push

---

## 산출물 요약

### 신규/수정 파일
- 24개 신규 spec 파일 (8 screens × 3)
- 6개 기존 spec 파일 업데이트
- 1개 인덱스 갱신 (`spec/README.md`)
- 3개 리뷰 (`review-*.md`)
- 2개 계획/결정 (`plan.md`, `decisions.md`)
- 1개 진행 기록 (이 파일)

### 깃 커밋 히스토리
1. `c59337b` — 베이스라인 (spec 10개 / docs/2026-05-11 / 3인 리뷰)
2. `965ac04` — plan.md + decisions.md 초안
3. `a326c9e` — Q-12-01~12 사용자 확정 (추천안 채택)
4. (예정) — Phase 1: 스텁 8개 화면 보강 + decisions §D 적용

---

## Phase 2 — 신규 5개 화면 작성 완료

### 신규 작성 (20개 파일)
| 화면 | overview | ui | interactions | data |
|---|---|---|---|---|
| report | ✅ | ✅ | ✅ | ✅ |
| group-settings | ✅ | ✅ | ✅ | ✅ |
| blocked-users | ✅ | ✅ | ✅ | ✅ |
| account-delete | ✅ | ✅ | ✅ | ✅ |
| legal | ✅ | ✅ | ✅ | ✅ |

전체 화면: **15/15 (60문서)** 완성

### 추가 결정 (사용자 추가 요구)
- **D-12-13 영상 촬영 방향 — Landscape 전용** (사용자 요구로 추가)
  - 1280×720 16:9 고정
  - portrait 미지원 — 디바이스 회전 잠금 + 회전 안내 화면
  - Cloud Function이 메타 검증으로 portrait 영상 업로드 거부
  - 영향: capture/* 전체, group-detail/ui.md, chat 영상 인용 카드

## 미해결 / 내일 진행 표시

### Phase 3 사용자 영역 (병렬)
- Firebase 프로젝트 2개(dev/prod) 생성 + 서울 리전
- Apple Developer / Google Play 등록
- 이용약관 / 개인정보처리방침 문서 작성
- App Check 활성화

### Phase 4 (Phase 1~3 완료 후)
- Expo 프로젝트 스캐폴딩
- Firestore Security Rules 작성
- Cloud Functions 구현
- Firebase Emulator Suite 환경 구성
- k6 부하 테스트 시나리오 구성

---

## 학습 / 회고

### 잘된 점
- 3인 에이전트 병렬 리뷰로 단일 시점에서 다각도 검증 확보
- 3명 모두 동일하게 강조한 P0 5개를 명확히 추출 → 우선순위 흔들림 없음
- 사용자 결정 12개를 추천안과 함께 한 번에 제시 → 답변 단순화

### 개선 필요
- 에이전트 Write 권한 거부 이슈로 콘텐츠 텍스트 반환 후 메인 에이전트 작성 패턴 사용
  - 토큰 비용 + conversation 길어짐
  - 다음에는 처음부터 "콘텐츠만 반환" 방식으로 프롬프트 작성
- PAT가 채팅 노출됨 → 사용자에게 즉시 revoke 권고 (보안)
- 깃 저장소 루트가 홈 디렉토리에 있던 문제 → 작업 시작 전 검증 필요

### 기록할 만한 결정
- timezone은 UTC 저장 + 로컬 표시 (다중 timezone 그룹 안전)
- 안읽음 카운트는 클라이언트 계산 (Firestore 핫 쓰기 회피)
- App Check Day-1 적용 (SMS toll fraud 방어)
- 만 14세 이상만 가입 (한국 KISA 우선)
