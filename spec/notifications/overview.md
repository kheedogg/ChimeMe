# notifications — 알림 설정

## 화면 키
`notifications`

## 진입로
- 프로필 → 알림

## 목적
- 푸시 알림 종류별 ON/OFF 및 시간대 설정

## 사용자 시나리오
1. 영상 알림 (매시 정각 트리거) ON/OFF
2. 채팅 알림 ON/OFF
3. 방해 금지 시간 설정 (예: 23:00 ~ 07:00 알림 차단)

## 확정 정책 (2026-05-12, Q-12-09)
- **기본값**:
  - 전체 알림: ON
  - 영상 알림: ON
  - 채팅 알림: ON
  - 방해 금지 시간: ON (23:00 ~ 07:00, KST)
  - 다이제스트 모드: OFF (3개 이상 그룹 가입 시 옵션 노출)
- **시간 picker UX**: 네이티브 시간 picker (`UIDatePicker` / `TimePickerDialog`)
- **그룹별 세부 설정**: 지원 (`memberships.notificationsEnabled.video / chat`)
- **다이제스트 모드**:
  - 3개 이상 그룹 가입 시 옵션 노출
  - ON 시 정각 알림을 단일 알림으로 통합 ("N개 그룹에서 새 시간이 시작됐어요")
- **시스템 권한 거부 시**: 상단 배너 + "설정 열기" 버튼
- **FCM 발송 시 체크 순서** (Cloud Function — D-12-06):
  1. `users.notifications.enabled` (전체)
  2. `users.notifications.video / chat` (유형별)
  3. `users.notifications.quietHours*` (시간대)
  4. `memberships.notificationsEnabled.video / chat` (그룹별)
  5. 모두 통과 시 토픽 메시지 발송 (jitter 0~120초 포함)

## 후속 작성 완료
- `ui.md` ✓ `interactions.md` ✓ `data.md` ✓
