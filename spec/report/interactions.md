# report — Interactions

## 진입
- params: `{ targetType: 'video' | 'message' | 'user', targetId: string }`
- 트리거: 영상/메시지 길게 누르기 + ActionSheet "신고하기" 선택, 또는 프로필 화면 신고 버튼

## 진입 시 사전 체크
1. **신고 기능 제한 체크** — `users/{uid}.reportBlocked` 또는 `users/{uid}.reportBlockedUntil > now`
   - 제한 중이면 24시간 제한 화면 표시
2. **본인 콘텐츠 신고 차단** — 작성자가 본인이면 ActionSheet에 "신고" 옵션 자체 미노출
3. **이미 신고한 콘텐츠** — `reports` 컬렉션에서 동일 `reporterId + targetId` 조회 → 존재 시 "이미 신고하셨어요" 안내

## 사유 선택
- 라디오 그룹: 단일 선택만 가능
- 선택 변경 시 즉시 UI 갱신
- "기타" 선택 시 부연 설명 영역 강조 (테두리 + 라벨 빨강)

## 부연 설명 입력
- 일반 사유: 선택 입력 (없어도 제출 가능)
- "기타" 사유: 필수 입력 (1자 이상)
- 500자 제한, 초과 시 입력 차단
- 한글 / 영문 / 이모지 / 줄바꿈 허용

## "신고하기" 제출
1. CTA 비활성 + 스피너
2. Firestore `reports/{reportId}` 문서 생성:
   ```typescript
   {
     id: reportId,
     targetType,
     targetId,
     reporterId: currentUser.uid,
     reason: 'profanity' | 'inappropriate' | 'spam' | 'violence' | 'minor_protection' | 'other',
     description: descriptionText || null,
     createdAt: serverTimestamp(),
     status: 'pending', // pending | reviewed | dismissed
   }
   ```
3. **즉시 가림 처리** (클라이언트):
   - 신고 대상이 영상이면 → 메모리에 신고된 video ID 추가 → group-detail에서 placeholder 표시
   - 신고 대상이 메시지면 → 메모리에 신고된 message ID 추가 → chat에서 placeholder 표시
   - 신고 대상이 사용자면 → 자동으로 `blocks/{myUid}_{theirUid}` 생성 (Q-12-04: 사용자 신고 = 차단 포함)
4. Cloud Function 트리거 (server-side):
   - 동일 `targetId` 신고 건수 카운트
   - 3건 이상 누적 시 `hiddenByReports: true` 설정 (영상 / 메시지)
   - 신고 사용자 본인 신고 카운터 갱신 (`users/{uid}.reportCount += 1`)
   - 24시간 이내 5건 이상이면 `reportBlockedUntil = now + 24h`

## 완료 흐름
1. 성공 응답 받으면 신고 완료 화면 전환
2. 사용자가 "확인" 탭 → 모달 닫힘
3. 진입 트리거 화면 (chat / group-detail)로 자동 복귀
4. 신고 대상 콘텐츠는 placeholder로 즉시 가려진 상태로 표시

## 사용자 신고 (특수 처리)
- `targetType === 'user'`일 때:
  - 신고와 동시에 자동 차단 처리 → `blocks/{myUid}_{theirUid}` 생성
  - 추가 안내: "이 사용자가 차단되었습니다. 차단 목록에서 해제할 수 있어요"
  - 그룹 내 해당 사용자의 모든 영상/메시지가 신고자 화면에서 가려짐

## 미성년자 보호 위반 신고
- 사유 "미성년자 보호 위반" 선택 시:
  - 즉시 graph 가림 처리 (3건 미만이어도)
  - Cloud Function이 관리자 알림 채널(Slack)로 즉시 발송
  - 관리자가 빠르게 검토 가능하도록 우선순위 표시

## 신고 후 진행 상황
- (옵션, 향후) 관리자가 검토 완료 시 푸시 알림:
  - "신고하신 콘텐츠가 그룹에서 가려졌습니다" 또는
  - "신고하신 콘텐츠는 가이드라인 위반에 해당하지 않습니다"
- 베타에서는 알림 없음 (P2)

## 24시간 제한 (허위 신고 패널티)
- 진입 시 `users/{uid}.reportBlockedUntil > now`이면 제한 화면 표시
- 남은 시간 카운트다운 (초 단위 실시간 업데이트)
- 0초 도달 시 자동으로 신고 화면 진입 가능

## 에러 처리
- 네트워크 오류 → "잠시 후 다시 시도해 주세요" 토스트, 입력 데이터 보존
- 이미 신고함 → 다이얼로그 "이미 신고하셨어요" + 모달 닫기

## 접근성
- 사유 선택 변경 a11y announce
- 제출 진행 상태 a11y live region
- 완료 / 실패 결과 a11y assertive

## 트래킹
- `report_view` (targetType)
- `report_submit` (targetType, reason)
- `report_blocked_view` (24시간 제한 진입)
- `report_already_reported` (중복 신고 시도)
- `report_threshold_reached` (3건 누적 가림)
