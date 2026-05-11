# group-settings — Interactions

## 진입
- params: `groupId`
- 진입 시 그룹 정보 + 멤버 정보 + 초대 링크 정보 fetch

## 그룹 정보 편집 (소유자만)

### 이름 변경
1. 편집 아이콘 탭 → 텍스트 필드 활성
2. 1~30자 검증
3. "저장" → `groups/{gid}.name` 갱신
4. 그룹 채팅에 시스템 메시지: "그룹 이름이 'X'로 변경됐어요"

### 이모지 변경
1. 이모지 카드 탭 → 이모지 picker 모달
2. 새 이모지 선택 → `groups/{gid}.emoji` 갱신
3. 시스템 메시지 발송

## 초대 링크

### 복사
- "복사" 탭 → 클립보드에 URL 복사 + 토스트

### 새로 만들기 (소유자만)
1. 기존 `invites` 문서 만료 처리 (`expiresAt = now`)
2. 새 `invites` 문서 생성 (7일 유효)
3. UI 갱신

## 멤버 관리

### 멤버 항목 탭
- 일반 멤버 탭: 해당 사용자 profile 미니뷰 또는 신고 메뉴
- 소유자가 다른 멤버 길게 누르기 → ActionSheet:
  - 프로필 보기
  - 신고하기 → `report` 화면 진입
  - **그룹에서 내보내기** (소유자만)

### 멤버 추방 (소유자만)
1. ActionSheet → "그룹에서 내보내기" 선택
2. 확인 다이얼로그 표시 (재가입 차단 옵션 체크박스)
3. "내보내기" 탭 → 처리:
   - Firestore 트랜잭션:
     - `groups/{gid}.memberIds`에서 해당 uid 제거
     - `memberships/{kickedUid}_{gid}` 삭제
     - (체크박스 ON시) `blockedFromGroups/{kickedUid}_{gid}` 생성 (재가입 차단)
   - Cloud Function 트리거:
     - 추방자 FCM 토픽 (`group_{gid}`) 구독 해제
     - 추방자에게 푸시 알림: "[그룹명]에서 내보내졌습니다"
   - 그룹 채팅에 시스템 메시지: "민지님이 그룹에서 내보내졌어요"
4. 멤버 리스트 갱신

## 본인 탈퇴 (일반 멤버)
1. "그룹 나가기" 탭 → 확인 다이얼로그
2. 확인 → 처리:
   - `groups/{gid}.memberIds`에서 본인 uid 제거
   - `memberships/{myUid}_{gid}` 삭제
   - FCM 토픽 구독 해제 (`group_{gid}`)
   - 본인의 채팅 메시지 `authorId` 익명화 → "(탈퇴한 사용자)" 표시
   - 본인의 영상은 잔존 (Q-12-06)
3. 시스템 메시지: "민지님이 그룹을 떠났어요"
4. 메인 화면으로 자동 이동 (해당 그룹 카드 제거)

## 권한 위임 (소유자)

### 1단계: 멤버 선택
1. "권한 위임하기" 탭 → 멤버 선택 화면
2. 본인 제외 멤버 리스트
3. 한 명 선택 → "다음" 활성

### 2단계: 확인
1. 확인 탭 → 처리:
   - 트랜잭션:
     - `groups/{gid}.ownerId = newOwnerUid`
     - `memberships/{newOwnerUid}_{gid}.role = 'owner'`
     - `memberships/{myUid}_{gid}.role = 'member'`
   - 시스템 메시지: "민지님이 그룹 소유자가 되었어요"
2. 현재 화면 갱신 (본인 권한 제거된 일반 멤버 뷰)

## 소유자 탈퇴 (Q-12-06 강제 선택)
- 소유자가 그룹 나가기 시도 시:
  - 일반 다이얼로그 대신 **3가지 선택 화면**:
    1. "그룹 해체" → 그룹 해체 흐름
    2. "권한 위임 후 탈퇴" → 권한 위임 흐름 + 자동 탈퇴
    3. "취소"

## 그룹 해체 흐름 (소유자)

### 1단계: 안내
- "취소" / "다음" 분기

### 2단계: 그룹 이름 입력 확인
- 정확한 그룹 이름 입력 시 CTA 활성
- "그룹 영구 해체" 탭 → 처리:
  1. CTA 비활성 + 스피너
  2. Cloud Function `dissolveGroup` 호출:
     - 그룹 멤버 전원에게 FCM 알림 발송 (그룹 해체)
     - `groups/{gid}/posts/*` 일괄 삭제 + Storage 영상 일괄 삭제
     - `groups/{gid}/messages/*` 일괄 삭제
     - `memberships` where groupId 일괄 삭제
     - `invites` where groupId 일괄 삭제
     - 모든 멤버의 FCM 토픽 구독 해제
     - `groups/{gid}` 문서 삭제
  3. 완료 안내 토스트
  4. 메인 화면으로 강제 이동

## 차단된 사용자 재초대 (Q-12-06)
- 초대 링크로 진입 시 `blockedFromGroups/{uid}_{gid}` 체크
- 존재 시 가입 차단 + 안내: "이 그룹에 가입할 수 없어요"

## 권한 분기 UI
- 본인이 소유자가 아니면:
  - 편집 아이콘 미노출
  - "권한 위임"/"그룹 해체" 미노출
  - 멤버 ⋮ 메뉴에서 "내보내기" 제거
  - "새로 만들기" 초대 링크 버튼 비활성

## 에러 처리
- 추방 실패 → "잠시 후 다시 시도해 주세요" 토스트
- 그룹 해체 실패 → 단계별 재시도 가능 (idempotent)

## 실시간 갱신
- 그룹 정보 / 멤버 변경은 `onSnapshot`으로 실시간 반영
- 본인이 추방당하면 자동으로 메인 이동 (멤버십 삭제 감지)

## 접근성
- 소유자 권한 변경 a11y assertive
- 위험 액션 안내 a11y
- 진행 단계 a11y live region

## 트래킹
- `group_settings_view`
- `group_name_changed`
- `group_emoji_changed`
- `group_invite_refreshed`
- `group_member_kicked` (with block option)
- `group_member_left` (본인 탈퇴)
- `group_ownership_transferred`
- `group_dissolved`
