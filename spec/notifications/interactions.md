# notifications — Interactions

## 진입
- profile 화면 메뉴 "알림 설정" → `notifications` 라우트
- 진입 시:
  1. `users/{uid}.notifications` fetch (또는 캐시)
  2. 가입 그룹 수 fetch → 다이제스트 모드 노출 여부 결정
  3. 시스템 알림 권한 상태 확인 (`Notifications.getPermissionsAsync()`)

## 전체 알림 토글
- ON ↔ OFF
- `users/{uid}.notifications.enabled` 갱신
- OFF 시:
  - 모든 세부 옵션 disabled (시각적으로만; 값은 보존)
  - Cloud Function이 발송 직전 `users.notifications.enabled === false`면 전체 알림 발송 차단

## 영상 알림 토글
- `users/{uid}.notifications.video: bool` 갱신
- Cloud Function 매시 정각 발송 로직에서 이 값 체크

## 채팅 알림 토글
- `users/{uid}.notifications.chat: bool` 갱신
- 채팅 메시지 발송 시 FCM 알림 발송 직전 이 값 체크

## 다이제스트 모드 (Q-12-09)
- 3개 이상 그룹 가입자에게만 토글 노출
- ON 시:
  - 정각 알림이 그룹별 개별 발송 → 단일 다이제스트로 묶임
  - 알림 내용: "**N개 그룹**에서 새 시간이 시작됐어요"
  - 탭 시 메인 화면으로 이동
- OFF 시: 그룹별 개별 알림 (기본 동작)

## 방해 금지 시간

### 토글 ON
- 시간 picker 활성
- 시작/종료 시간 변경 시 즉시 Firestore 갱신

### 시간 picker
- 네이티브 시간 picker (iOS `UIDatePicker`, Android `TimePickerDialog`)
- 분 단위 정확도 (15분 단위로 제한할지 검토)
- 시작이 종료보다 늦은 경우 (예: 22:00 → 06:00) → 다음 날까지 적용 (over-midnight)

### Cloud Function 발송 시 체크
- 현재 사용자의 timezone(`users.timezone`) 기준 현재 시각이 quiet hours 범위 내인지 체크
- 범위 내면 FCM 발송 건너뛰기

## 그룹별 알림 설정 (서브 화면)
- 진입 시 `memberships` where `uid == currentUser.uid` 전체 fetch + 각 그룹 정보 join
- 그룹별 토글:
  - 영상 알림: `memberships/{uid}_{gid}.notificationsEnabled.video`
  - 채팅 알림: `memberships/{uid}_{gid}.notificationsEnabled.chat`
- 토글 OFF 시 FCM 토픽 구독은 유지하되 클라이언트가 무시 (서버 측 발송에서도 체크)

## 시스템 권한 상태
- 진입 시 권한 상태 확인
- "허용됨" → 정상 표시
- "거부됨" → 빨간 배너 + "설정 열기" 버튼
  - 탭 시 `Linking.openSettings()` → 시스템 알림 설정 → 변경 후 복귀
  - `AppState` 변경 감지하여 권한 재체크 → 자동 갱신

## 변경 즉시 반영 (낙관적 업데이트)
1. 토글 시각 갱신 (UI 우선)
2. Firestore 비동기 갱신
3. 실패 시 토글 원상 복귀 + 토스트 "변경 실패. 다시 시도해 주세요"

## 알림 권한 미허용 처리
- 시스템 알림 권한 거부 시:
  - 모든 클라이언트 토글은 활성으로 유지 (사용자 의지 보존)
  - 그러나 실제 알림은 표시 안 됨
  - 상단 배너로 시스템 권한 거부 상태 명시

## 접근성
- 토글 변경 시 a11y announce: "영상 알림 켜짐" / "영상 알림 꺼짐"
- 시간 picker 변경 시 a11y: "시작 시간 23시 0분으로 변경됨"
- 시스템 권한 상태 a11y assertive

## 트래킹
- `notifications_view`
- `notifications_master_toggle` (on/off)
- `notifications_video_toggle`
- `notifications_chat_toggle`
- `notifications_digest_toggle`
- `notifications_quiet_hours_changed` (start/end)
- `notifications_group_toggle` (groupId, type)
- `notifications_system_permission_opened`
