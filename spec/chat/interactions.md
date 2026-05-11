# chat — Interactions

## 진입
- params: `groupId`
- 그룹 상세 헤더 채팅 아이콘 또는 푸시 알림 딥링크로 진입
- 진입 시:
  1. `groups/{groupId}/messages` `onSnapshot` 구독 (최신 50건)
  2. `memberships/{uid}_{groupId}.lastReadMessageAt = serverTimestamp()` 갱신
  3. 메시지 입력 영역 포커스는 자동 X (사용자 의지 존중)

## 메시지 수신
- `onSnapshot`으로 실시간 갱신
- 추방/그룹 해체 감지: `groups/{gid}.memberIds`에서 본인 제외 시 → 메시지 발송 차단 + 안내 모달 + 메인으로 이탈
- 차단된 사용자의 메시지는 클라이언트에서 필터링 (Q-12-04 — 차단 시 기존 메시지도 가림)

## 메시지 발송
1. 입력 텍스트 검증:
   - 빈 텍스트 + 빈 인용 → 전송 비활성
   - 텍스트 1자 이상 OR 영상 인용 존재 → 전송 활성
2. 전송 탭:
   - 클라이언트 임시 ID 생성 (`clientMessageId = uuid()`)
   - 메시지 리스트에 낙관적 추가 (`status: 'sending'`)
   - Firestore `setDoc(doc(db, 'groups', gid, 'messages', clientMessageId), {...})` (D-12-08 멱등성)
3. 성공 → `status: 'sent'`로 갱신
4. 실패 → `status: 'failed'` 빨간 인디케이터 + 재시도 버튼
5. 재시도 → 동일 `clientMessageId`로 setDoc (멱등성으로 중복 방지)

## 영상 인용 선택
1. + 버튼 → ActionSheet → "이전 영상 인용"
2. 슬롯 picker 모달 표시:
   - 그룹의 최근 7일 슬롯 리스트
   - 각 슬롯 내 멤버 영상 썸네일 그리드 (8명)
3. 영상 탭 → 입력 영역 위에 인용 카드 미리보기
4. 사용자가 텍스트 추가 가능
5. 전송 시 메시지 + 인용 (`videoQuote: { postId, slotKey, authorId }`) 함께 저장

## 이모지 추가
- 입력 텍스트에 이모지 삽입 (시스템 기본 이모지 키보드)
- 별도 이모지 picker는 옵션 (P2)

## 메시지 삭제 (Q-12-05 — 5분 제한)
1. 본인 메시지 길게 누르기 → ActionSheet
2. 5분 이내: "삭제" 활성
3. 5분 초과: "삭제" 비활성 + 회색
4. 삭제 탭 → 확인 다이얼로그 → Firestore `update`:
   ```
   { content: null, deleted: true, deletedAt: serverTimestamp() }
   ```
5. UI에서 "삭제된 메시지" placeholder로 갱신

## 메시지 신고 (Q-12-04)
1. 타인 메시지 길게 누르기 → ActionSheet → "신고하기"
2. 신고 사유 선택 모달:
   - 욕설/혐오
   - 음란물/부적절
   - 스팸/광고
   - 기타
3. 사유 선택 → `reports/{reportId}` 생성 + 신고자 화면에서 즉시 가림
4. 동일 메시지 누적 3건 신고 시 그룹 전체 가림 + 시스템 메시지 노출

## 사용자 차단 (Q-12-04)
1. 타인 메시지 길게 누르기 → "사용자 차단"
2. 확인 다이얼로그: "[멤버명]을 차단하시겠어요? 차단 후 이 사람의 메시지와 영상이 보이지 않아요"
3. 차단 → `blocks/{myUid}_{theirUid}` 생성
4. 클라이언트 즉시 해당 사용자 메시지 필터링 + 그룹 상세 영상도 가림
5. 차단 해제는 `profile` 화면의 차단 목록에서 가능

## 페이지네이션
- 메시지 리스트 상단 도달 시 50건 추가 로드 (`startAfter` 커서)
- 로딩 인디케이터 상단 표시

## "모두 읽음" 판정 (D-12-08)
- 헤더에 "모두 읽음" / "안 읽은 멤버 N명" 표시
- 클라이언트가 그룹 멤버들의 `memberships/{*_gid}.lastReadMessageAt`을 읽어 비교
- 가장 최신 메시지의 `createdAt`보다 모든 멤버의 `lastReadMessageAt`이 같거나 더 크면 "모두 읽음"

## 키보드 / 포커스
- 입력 포커스 시 메시지 리스트 자동 하단 스크롤
- 포커스 해제 (외부 탭) 시 키보드 닫힘

## 오프라인 처리
- Firestore 오프라인 캐시 활성 (`enablePersistence`)
- 발송한 메시지는 로컬에 큐잉 후 온라인 복귀 시 자동 동기화
- UI에 "전송 중..." 상태 표시

## 푸시 알림 (수신)
- 채팅 알림은 영상 알림과 별도 ON/OFF (`memberships.notificationsEnabled.chat`)
- 메시지 수신 시 토픽 메시지: `group_{gid}_chat`
- 본인이 채팅 화면에 활성 상태일 때는 FCM 토큰을 별도 처리하여 인앱에서만 표시

## 에러 처리
- 발송 실패 → 메시지 빨간 인디케이터 + 재시도 / 삭제 옵션
- 권한 잃음 (강퇴) → 모달 안내 + 자동 메인 복귀

## 접근성
- 신규 메시지 도착 a11y live region
- 메시지 액션 시트 a11y 각 옵션 명시
- 영상 인용 a11y: "민지의 09:00 슬롯 영상이 인용된 메시지"

## 트래킹
- `chat_view`
- `chat_message_sent` (text / video_quote / both)
- `chat_message_deleted`
- `chat_message_reported`
- `chat_user_blocked`
- `chat_video_quote_tapped` (인용 카드 탭으로 슬롯 점프)
