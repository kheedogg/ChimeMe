# profile — Interactions

## 진입
- 메인 우상단 프로필 아이콘 → `profile` 라우트

## 사진 변경
1. 아바타 또는 편집 아이콘 탭 → ActionSheet
2. "카메라로 촬영": `expo-image-picker` 카메라 (권한 필요)
3. "갤러리에서 선택": `expo-image-picker` 라이브러리 (권한 필요)
4. "기본 이미지로 변경": Storage 파일 삭제 + `users/{uid}.photoUrl = null`
5. 선택 후 크롭 (1:1) → 업로드 → `users/{uid}.photoUrl` 갱신

## 닉네임 편집
1. 편집 아이콘 탭 → 텍스트 필드 활성 + 키보드 노출
2. 검증: 1~20자, 부적절 단어 (onboarding과 동일)
3. "저장" → `users/{uid}.displayName` 갱신 + 그룹 멤버에 자동 반영 (Firestore 참조 갱신)
4. "취소" → 원래 닉네임 복원

## 메뉴 진입
| 항목 | 동작 |
|---|---|
| 알림 설정 | `notifications` 라우트로 이동 |
| 차단한 사용자 | `blocked-users` 라우트 |
| 내 그룹 | (옵션) 그룹 리스트 — 또는 메인으로 |
| 도움말 | 외부 브라우저로 도움말 페이지 |
| 이용약관 | 외부 브라우저로 약관 페이지 |
| 개인정보처리방침 | 외부 브라우저 |
| 버전 정보 | 탭 5회 → 디버그 메뉴 (개발자 옵션) |

## 차단 목록 (blocked-users)
- 진입 시 `blocks` 컬렉션에서 본인이 차단한 사용자 fetch
- 각 항목 "차단 해제" 탭:
  - 확인 다이얼로그: "[닉네임]의 차단을 해제하시겠어요?"
  - 확인 → `blocks/{myUid}_{theirUid}` 삭제
  - 차단 해제된 사용자의 기존 메시지/영상 즉시 다시 표시

## 로그아웃
1. "로그아웃" 탭 → 확인 다이얼로그
2. 확인:
   - FCM 토큰 제거 (`users/{uid}.fcmTokens`에서 현재 디바이스 토큰 제거)
   - 모든 그룹 토픽 구독 해제
   - Firebase Auth `signOut()`
   - secure storage 클리어
   - `auth-phone`으로 라우팅 (replace)

## 계정 삭제 흐름 (Q-12-06)

### 1단계: 안내 진입
1. "계정 삭제" 탭 → 1단계 화면
2. 사용자가 "취소" 탭 → profile 복귀
3. "다음" 탭 → 소유 그룹 확인

### 2단계: 소유 그룹 처리
- 본인이 `ownerId`인 그룹 fetch
- 각 그룹마다 사용자가 선택:
  - **그룹 해체**:
    - 안내: "이 그룹은 영구 삭제됩니다. 모든 영상/채팅이 사라집니다"
    - 확인 → 다음 진행 시 일괄 삭제 처리
  - **권한 위임**:
    - 그룹 멤버 리스트에서 새 소유자 선택
    - 선택 후 확인 다이얼로그
- 모든 그룹 처리 완료 시 "다음" 활성
- 소유 그룹 0개면 이 단계 스킵

### 3단계: 최종 확인
1. "삭제" 정확히 입력 → CTA 활성
2. 삭제 탭 → 처리 시작:
   - 진행 표시 화면 (스피너 + "1/4: 그룹 정리 중..." 등)
3. Cloud Function `deleteAccount` 호출:
   - 본 사용자가 멤버인 모든 그룹에서 본인 제거
   - 본인 영상 모두 Storage 삭제
   - 메시지 `authorId` 익명화
   - `users/{uid}` 삭제
   - Firebase Auth 계정 삭제
   - FCM 토큰 일괄 폐기 + 토픽 구독 해제
4. 완료 → "계정이 삭제됐어요. ChimeMe를 이용해 주셔서 감사합니다" 안내 → `auth-phone`으로 강제 복귀

### 삭제 도중 오류
- 부분 실패 시 재시도 또는 관리자 문의 안내
- 트랜잭션 단위로 처리하여 중간 상태 방지 (가능한 범위 내)

## 권한 위임 시
- 새 소유자에게 시스템 메시지 자동 발송 (그룹 채팅에)
- 새 소유자의 `memberships.role` → `owner`로 변경
- 기존 소유자의 `memberships` 삭제 (계정 삭제 일부)

## 그룹 해체 시 (Q-12-06)
- 모든 멤버에게 FCM 알림: "[그룹명]이 해체되었습니다"
- 그룹의 모든 영상 / 채팅 / memberships / invites Cloud Function 일괄 삭제

## 에러 처리
- 닉네임 갱신 실패 → 토스트 + 원래 값 복원
- 사진 업로드 실패 → 토스트 + 이전 사진 유지
- 로그아웃 실패 → 에러 다이얼로그 + 재시도 옵션
- 계정 삭제 실패 → 단계별 재시도 가능 (idempotent 처리)

## 접근성
- 위험 액션은 a11y assertive announce
- 진행 단계 a11y live region
- 인라인 편집 모드 진입/이탈 a11y announce

## 트래킹
- `profile_view`
- `profile_nickname_changed`
- `profile_photo_changed` (source: camera / gallery / reset)
- `profile_logout`
- `account_delete_start` / `_step1` / `_step2` / `_step3` / `_complete` / `_cancelled` / `_error`
- `group_dissolve` (계정 삭제 흐름에서) / `group_ownership_transferred`
- `user_unblocked`
