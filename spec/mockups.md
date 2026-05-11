# ChimeMe — 화면 목업 인덱스

각 화면의 디자인 목업 보유 현황과 작업 의뢰 트래킹.

## 보유 목업

| 화면 | 파일 | 상태 |
|---|---|---|
| `main` | [`AppPlanningDoc/pages/mainPage.png`](../AppPlanningDoc/pages/mainPage.png) | ✅ 1차 |
| `group-detail` (활성 슬롯) | [`AppPlanningDoc/pages/groupPage.png`](../AppPlanningDoc/pages/groupPage.png) | ✅ 1차 |
| `group-detail` (미업로드 슬롯) | [`AppPlanningDoc/pages/groupPage2.png`](../AppPlanningDoc/pages/groupPage2.png) | ✅ 1차 |

## 누락 — 디자인 필요 화면 (8개)

각 화면당 권장 추가 시안.

| 화면 | 필수 시안 | 추가 권장 |
|---|---|---|
| `auth-phone` | 입력 + 약관 체크박스 | — |
| `auth-verify` | OTP 6박스 + 카운트다운 | — |
| `onboarding` | 스텝1 생년월일 / 스텝2 닉네임+사진 | 만 14세 미만 차단 화면 |
| `group-create` | 모달 시트 (이름+이모지+초대 액션) | QR 풀스크린 모달 |
| `capture` | 카메라 미리보기 + 카운트다운 + 미리보기 | 권한 거부 화면 / 슬롯 마감 화면 |
| `chat` | 메시지 리스트 + 입력 영역 | 영상 인용 카드 / 메시지 액션시트 |
| `profile` | 프로필 카드 + 메뉴 섹션 | 계정 삭제 3단계 흐름 |
| `notifications` | 토글 리스트 + 시간 picker | 그룹별 알림 서브 화면 |

## 디자인 의뢰 가이드

각 화면의 상세 레이아웃·컴포넌트·디자인 토큰은 `spec/<screen>/ui.md` 참조.
디자이너에게 의뢰 시 같이 전달할 자료:

1. **컬러 / 폰트**: `spec/README.md` "디자인 시스템" 섹션
2. **화면 상세**: `spec/<screen>/ui.md` 풀 텍스트
3. **인터랙션 흐름**: `spec/<screen>/interactions.md`
4. **참고 앱**: `docs/2026-05-11/photos/` (SetLog 스크린샷)
5. **확정된 정책**: `docs/2026-05-12/decisions.md`

## 파일명 규칙 (디자인 완성 후)

```
AppPlanningDoc/pages/<screen-key>[-<variant>].png
```
예시:
- `auth-phone.png` (기본)
- `auth-phone-error.png` (에러 상태)
- `capture-countdown.png` (카운트다운 중)
- `capture-permission-denied.png` (권한 거부)

추가 시 본 인덱스(`spec/mockups.md`)와 해당 `ui.md` 상단에 임베드.

## 디자인 시스템 토큰 (확정)

| 토큰 | 값 |
|---|---|
| `color.primary` | `#FFD54F` 톤 노란색 |
| `color.surface` | `#FFFFFF` |
| `color.surfaceMuted` | 옅은 그레이 (`#F5F5F5` 톤) |
| `color.textPrimary` | 짙은 그레이/블랙 |
| `color.textMuted` | 중간 그레이 |
| `color.error` | `#E53935` |
| `radius.card` | 16 |
| `radius.fab` | 999 (원형) |
| `font.heading` | 굵은 sans-serif |
