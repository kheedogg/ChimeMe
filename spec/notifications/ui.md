# notifications — UI

## 레이아웃

### 1. 헤더
- 좌측 ← 뒤로가기 (profile로)
- 가운데: **알림 설정**

### 2. 전체 알림 ON/OFF (최상단)
```
🔔  알림 받기                       [● ─ ○]
    꺼두면 모든 알림이 차단돼요
```
- 토글 OFF 시 하단 모든 세부 옵션 비활성 + 회색 처리

### 3. 알림 유형 섹션

#### 3.1 영상 알림
```
📹  매시 정각 영상 알림              [● ─ ○]
    그룹에 새 영상이 올라오면 알려드려요
```

#### 3.2 채팅 알림
```
💬  채팅 알림                       [● ─ ○]
    그룹 채팅 메시지를 알려드려요
```

#### 3.3 다이제스트 모드 (3개+ 그룹 가입자만 노출 — Q-12-09)
```
📦  다이제스트 모드                  [○ ─ ●]
    여러 그룹 알림을 하나로 묶어서 받아요
```

### 4. 방해 금지 시간 섹션
```
🌙  방해 금지 시간                  [● ─ ○]

    시작 시간    [23:00 ▼]
    종료 시간    [07:00 ▼]
    
    이 시간 동안은 알림이 표시되지 않아요
```
- 토글 OFF 시 시간 picker 비활성
- 시간 picker: 24시간 형식, 분 단위 미세 조절 (네이티브 picker)

### 5. 그룹별 알림 섹션 (메뉴)
```
👥  그룹별 알림 설정                  >
    그룹마다 알림 ON/OFF를 따로 설정할 수 있어요
```
- 탭 시 `notifications/groups` 서브 화면으로 이동:
  - 가입한 그룹 리스트
  - 각 그룹마다:
    - 그룹 이모지 + 이름
    - 영상 알림 토글
    - 채팅 알림 토글

### 6. 시스템 권한 영역 (하단)
```
⚙️  시스템 알림 권한                  허용됨
    또는
⚙️  시스템 알림 권한                  거부됨
    [설정에서 허용하기 →]
```
- iOS/Android 시스템 알림 권한 상태 표시
- 거부 상태면 "설정 열기" 버튼

## 디자인 토큰
- `notification.toggleColor.on` — `color.primary`
- `notification.toggleColor.off` — 회색
- `notification.disabledOpacity` — 0.4
- `notification.sectionHeader.color` — 짙은 회색

## 컴포넌트
- `NotificationToggleRow` — `{ icon, label, description, value, onChange, disabled? }`
- `TimeRangePicker` — `{ startTime, endTime, onChange, disabled? }`
- `GroupNotificationItem` — `{ group, videoEnabled, chatEnabled, onToggle }`
- `SystemPermissionStatus` — `{ status: 'granted' | 'denied' | 'undetermined', onOpenSettings }`

## 상태별 UI
- 전체 알림 OFF (모든 세부 옵션 회색)
- 방해 금지 ON (시간 picker 활성)
- 시스템 권한 거부 (상단 배너 + "설정 열기")
- 그룹 0개 (그룹별 설정 메뉴 비활성)

## 접근성
- 각 토글 a11y label + 현재 상태 announce
- 시간 picker a11y: "시작 시간 23시 0분 선택됨"
- 시스템 권한 상태 a11y assertive

## 기본값 (Q-12-09 확정)
- 전체 알림: ON
- 영상 알림: ON
- 채팅 알림: ON
- 다이제스트: 3개 이상 그룹 가입 시 노출, 기본 OFF
- 방해 금지: 23:00 ~ 07:00 (KST), 기본 ON

## 변경 즉시 반영
- 토글 변경 시 즉시 Firestore에 저장 (낙관적 업데이트)
- 실패 시 토글 원상 복귀 + 토스트
