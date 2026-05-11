# account-delete — Interactions

## 진입
- profile → "계정 삭제" 메뉴 → `account-delete/step1` 라우트
- 진입 즉시:
  1. 소유 그룹 fetch (`groups where ownerId == currentUser.uid`)
  2. 결과를 메모리에 보관

## 스텝 1 → 스텝 2 전환

### "취소" 탭
- 모달/화면 닫기 → profile 복귀
- 작성 중 데이터 없음 (메모리 클리어)

### "다음" 탭
- 소유 그룹 0개: 스텝 3로 직접 이동
- 소유 그룹 1개 이상: 스텝 2로 이동

## 스텝 2: 그룹 처리 선택

### 데이터 모델
```typescript
type OwnerGroupDecision = {
  groupId: string;
  action: 'dissolve' | 'transfer';
  newOwnerId?: string;  // action이 'transfer'일 때만
};
```

### 선택 검증
- 모든 그룹에 대해 `action`이 지정되어야 함
- `action === 'transfer'`이면 `newOwnerId`도 지정 필요
- 완료 시 "다음" 활성

### 권한 위임 멤버 선택
- 드롭다운 또는 모달 picker
- 멤버 리스트: 본인 제외 그룹 멤버
- 선택 시 즉시 미리보기 갱신

## 스텝 3: 최종 확인

### 입력 검증
- 정확히 "삭제" 텍스트만 활성화 트리거
- 영문 대소문자 무관 무시 (한글 "삭제" 정확히)
- 앞뒤 공백 자동 trim
- 빈 입력 / 다른 텍스트 → CTA 비활성

### "계정 영구 삭제" 탭
1. CTA 비활성 + 스피너
2. 네비게이션 차단 (백버튼/스와이프 막기)
3. Cloud Function `deleteAccount` 호출:
   ```typescript
   {
     ownerGroupDecisions: OwnerGroupDecision[]
   }
   ```
4. 진행 상태 화면으로 전환

## 처리 진행 흐름 (Cloud Function 응답 단계별)

### 단계 1: 소유 그룹 처리
- 각 그룹의 `action`에 따라 `dissolveGroup` 또는 `transferOwnership` 호출
- 진행 표시: "1/4 ✅ 그룹 정리 완료"
- 실패 시 단계 중단 + 에러 화면

### 단계 2: 본인 영상 일괄 삭제
- 본인이 멤버인 모든 그룹에서 `groups/*/posts where authorId == uid` fetch
- Storage 파일 일괄 삭제
- Firestore `posts` 문서 일괄 삭제
- 진행 표시: "2/4 ✅ 영상 삭제 완료"

### 단계 3: 채팅 메시지 익명화
- 본인이 멤버인 모든 그룹에서 `groups/*/messages where authorId == uid` fetch
- 일괄 `update({ authorId: 'deleted_{hash}' })` (메시지 내용 보존)
- 진행 표시: "3/4 ✅ 메시지 익명화 완료"

### 단계 4: 사용자 데이터 + 인증 삭제
- `memberships/{uid}_*` 전체 삭제
- `blocks` 양방향 삭제 (블록한/된 양쪽 정리)
- `profiles/{uid}.jpg` Storage 삭제
- `users/{uid}` 삭제
- Firebase Auth `deleteUser` 호출
- 진행 표시: "4/4 ✅ 계정 삭제 완료"

## 완료 처리
1. 클라이언트:
   - secure storage 클리어 (`auth.idToken`, `auth.session`)
   - AsyncStorage 클리어 (`upload_queue`, `notifications.draft` 등)
   - FCM 토큰 폐기
2. 완료 화면 표시
3. 5초 후 자동으로 `auth-phone`으로 이동 (replace 라우팅)
   - 또는 사용자가 "확인" 탭 시 즉시 이동

## 에러 처리

### 단계별 실패
- 단계 1 (그룹 처리) 실패 → "그룹 정리에 실패했어요. 다시 시도해 주세요"
  - 재시도 가능
- 단계 2~4 실패 → 단계까지는 완료, 다음 단계 재시도 가능
- 모든 단계 idempotent 처리 (재실행 안전)

### 부분 실패 화면
- 어떤 단계에서 실패했는지 명시
- "재시도" 버튼 (해당 단계부터 다시 시작)
- "고객센터 문의" 버튼 (외부 이메일 또는 폼)

### 네트워크 오류
- 처리 도중 네트워크 끊김 → "네트워크 연결 확인 후 다시 시도해 주세요"
- 일부 데이터가 이미 삭제됐을 수 있음 → 재시도 시 idempotent 보장

## 권한 위임 시 자동화
- 새 소유자에게 시스템 메시지 자동 발송 (그룹 채팅)
- 새 소유자에게 FCM 알림: "[그룹명]의 소유자가 되었습니다"

## 그룹 해체 시 자동화
- 멤버 전원에게 FCM 알림: "[그룹명]이 해체되었습니다"
- 멤버 전원의 그룹 토픽 구독 해제

## 백버튼 처리
- 스텝 1: profile로 복귀
- 스텝 2: 스텝 1로 (작성 중 데이터 유지)
- 스텝 3: 스텝 2로 (또는 스텝 1로 if 소유 그룹 0)
- 처리 중: 차단 (확인 다이얼로그 "처리 중에는 취소할 수 없어요")
- 완료/에러: 강제 이동 처리에 의해 백버튼 무의미

## 접근성
- 단계 전환 시 a11y assertive announce
- 위험 텍스트 a11y emphasize
- 진행 상태 a11y live region (각 단계 완료 시 announce)
- 처리 중 백버튼 차단 a11y 안내

## 트래킹
- `account_delete_step1_view`
- `account_delete_step1_cancel`
- `account_delete_step2_view` (owner group count)
- `account_delete_step2_decision` (action: dissolve/transfer)
- `account_delete_step3_view`
- `account_delete_submit`
- `account_delete_progress_step1_completed` / 2 / 3 / 4
- `account_delete_completed`
- `account_delete_error` (step where failed)
- `account_delete_retried`
