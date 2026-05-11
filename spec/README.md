# ChimeMe — 화면 기획서 인덱스

화면별 기획서는 각 폴더 내 4개 표준 문서로 구성한다:
`overview.md` · `ui.md` · `interactions.md` · `data.md`

## 화면 목록

| 키 | 한글명 | 상태 | 진입로 |
|---|---|---|---|
| [main](./main/overview.md) | 메인 (그룹 리스트) | 1차 작성 | 인증 후 진입 |
| [group-detail](./group-detail/overview.md) | 그룹 상세 (시간대별 영상) | 1차 작성 | 메인의 그룹 카드 탭 |
| [group-create](./group-create/overview.md) | 그룹 생성 · 초대 | 스텁 | 메인의 + FAB |
| [auth-phone](./auth-phone/overview.md) | 전화번호 입력 | 스텁 | 미인증 진입 |
| [auth-verify](./auth-verify/overview.md) | SMS 코드 확인 | 스텁 | 전화번호 입력 후 |
| [onboarding](./onboarding/overview.md) | 신규 사용자 온보딩 (닉네임/프로필) | 스텁 | 신규 인증 직후 |
| [capture](./capture/overview.md) | 3초 영상 촬영 | 스텁 | 그룹 상세 → 촬영 CTA |
| [chat](./chat/overview.md) | 그룹 채팅 | 스텁 | 그룹 상세 → 채팅 진입 |
| [profile](./profile/overview.md) | 내 프로필 | 스텁 | 메인 우상단 프로필 아이콘 |
| [notifications](./notifications/overview.md) | 알림 설정 | 스텁 | 프로필 → 알림 |

## 작성 규칙

- 화면 키는 kebab-case 영문
- 기획 변경 시 해당 화면 폴더의 문서를 직접 수정 (히스토리는 git이 관리)
- 실행/작업 진행 내역은 `docs/YYYY-MM-DD/`에 별도 기록 (`chimeme-screen-spec` skill 참고)

## 디자인 시스템

공통 디자인 토큰은 추후 `spec/design-system.md`에 분리 예정.
현재 확정 사항:
- Primary: `#FFD54F` 톤 노란색
- Surface: 화이트 + 옅은 그레이
- Heading: 굵은 sans-serif
- Card: 라운드 + 옅은 보더 + 미세 그림자

## 참고 자료

- 목업 이미지: `../AppPlanningDoc/pages/` (mainPage.png, groupPage.png, groupPage2.png)
- 데이터 모델 전체: `../docs/2026-05-11/decisions.md` 및 각 화면 `data.md`
