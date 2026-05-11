# ChimeMe — 화면 기획서 인덱스

화면별 기획서는 각 폴더 내 4개 표준 문서로 구성한다:
`overview.md` · `ui.md` · `interactions.md` · `data.md`

## 화면 목록

| 키 | 한글명 | 상태 | 진입로 |
|---|---|---|---|
| [main](./main/overview.md) | 메인 (그룹 리스트) | ✅ 1차 완성 | 인증 후 진입 |
| [group-detail](./group-detail/overview.md) | 그룹 상세 (시간대별 영상) | ✅ 1차 완성 | 메인의 그룹 카드 탭 |
| [group-create](./group-create/overview.md) | 그룹 생성 · 초대 | ✅ 1차 완성 | 메인의 + FAB |
| [auth-phone](./auth-phone/overview.md) | 전화번호 입력 | ✅ 1차 완성 | 미인증 진입 |
| [auth-verify](./auth-verify/overview.md) | SMS 코드 확인 | ✅ 1차 완성 | 전화번호 입력 후 |
| [onboarding](./onboarding/overview.md) | 신규 사용자 온보딩 (생년월일/닉네임/프로필) | ✅ 1차 완성 | 신규 인증 직후 |
| [capture](./capture/overview.md) | 3초 영상 촬영 | ✅ 1차 완성 | 그룹 상세 → 촬영 CTA |
| [chat](./chat/overview.md) | 그룹 채팅 | ✅ 1차 완성 | 그룹 상세 → 채팅 진입 |
| [profile](./profile/overview.md) | 내 프로필 | ✅ 1차 완성 | 메인 우상단 프로필 아이콘 |
| [notifications](./notifications/overview.md) | 알림 설정 | ✅ 1차 완성 | 프로필 → 알림 |

**완성도**: 10/10 화면 1차 완성 (각 4문서, 총 40개 문서) — 2026-05-12 Phase 1 완료

## 향후 작성 예정 (Phase 2)

| 키 | 한글명 | 상태 |
|---|---|---|
| `report` | 영상/메시지/사용자 신고 화면 | ⏳ |
| `group-settings` | 그룹 관리 (멤버 추방, 그룹 해체, 권한) | ⏳ |
| `blocked-users` | 차단 사용자 목록 | ⏳ |
| `account-delete` | 계정 삭제 흐름 (소유 그룹 처리) | ⏳ |
| `legal` | 이용약관 / 개인정보처리방침 | ⏳ |

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

## 핵심 정책 요약 (2026-05-12 확정)

| 항목 | 결정 |
|---|---|
| 슬롯 키 timezone | **UTC `YYYYMMDDHH` 저장 / 표시만 로컬** (D-12-01) |
| 업로드 grace window | 정각 + **30초** (D-12-02) |
| 영상 인코딩 | **720p H.264 / 30fps / 2~3Mbps / MP4** (D-12-03) |
| App Check | **Day-1 적용** (DeviceCheck / Play Integrity) (D-12-04) |
| 안읽음 카운트 | **클라이언트 계산** (`lastReadVideoAt` 기반, fan-out 제거) (D-12-05) |
| FCM 발송 | 토픽 메시징 + jitter 0~120초 + min instances (D-12-06) |
| 업로드 큐 | `expo-task-manager` + AsyncStorage + 지수 백오프 5회 (D-12-07) |
| 채팅 페이지네이션 | **50건/페이지** (D-12-08) |
| 그룹 최대 인원 | **8명** (2026-05-11) |
| 1인 최대 그룹 수 | **10개** (Q-12-01) |
| 영상 보존 기간 | **7일** (D-12-10) |
| 미성년자 정책 | **만 14세 이상만 가입** (Q-12-03) |
| 메시지 삭제 시간 | **5분** (Q-12-05) |
| 채팅 보존 기간 | **90일** (Q-12-08) |
| 푸시 기본값 | ON / 방해금지 23:00~07:00 / 3그룹+ 다이제스트 (Q-12-09) |
| 그룹 초대 | SMS 딥링크 + QR / 7일 유효 (Q-12-10) |
| 지원 국가 (베타) | **한국 only (+82)** (Q-12-11) |
| 약관 동의 | auth-phone 하단 체크박스 (필수/선택 분리) (Q-12-12) |

상세는 `docs/2026-05-12/decisions.md` 참조.

## 참고 자료

- 목업 이미지: `../AppPlanningDoc/pages/` (mainPage.png, groupPage.png, groupPage2.png)
- 데이터 모델 전체: `../docs/2026-05-12/decisions.md` 및 각 화면 `data.md`
- 3인 팀 리뷰: `../docs/2026-05-12/review-{planner,developer,tester}.md`
