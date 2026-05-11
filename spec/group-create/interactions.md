# group-create — Interactions

## 진입
- 메인 화면 우측 + FAB → 모달 바텀시트 오픈
- 시트 외부 탭 또는 ← 탭 → 닫기 (변경사항 있으면 확인 다이얼로그)

## 그룹 이름 검증
| 입력 | 동작 |
|---|---|
| 빈 값 | CTA 비활성 |
| 1~30자 | CTA 활성 |
| 31자+ | 입력 차단 |
| 부적절 단어 | 인라인 에러 |
| 앞/뒤 공백 | 자동 trim |
| 이모지 | 허용 |

## 이모지 picker
- 기본 이모지: 랜덤 (생성 시 자동 할당)
- 사용자가 변경 가능
- 이모지 카테고리: 스마일/음식/활동/장소/사물 등
- 검색 가능 (인접 입력 필드)

## "그룹 만들기" 탭
1. CTA 비활성 + 스피너
2. Firestore 트랜잭션 실행:
   - `groups/{newGroupId}` 문서 생성
   - `memberships/{uid}_{newGroupId}` 본인 멤버십 생성
   - `invites/{newInviteId}` 7일 유효 초대 링크 생성
3. FCM 토픽 구독: `group_{newGroupId}`
4. 모달 닫기
5. `group-detail/{newGroupId}`로 라우팅 (메인 위에 새 화면 push)

## 초대 링크 액션

### "SMS로 보내기"
1. 연락처 권한 요청 (`expo-contacts`)
2. 권한 허용 시: 연락처 picker → 다중 선택 가능
3. 권한 거부 시: "기본 SMS 앱 열기"로 fallback (메시지 본문에 링크 자동 입력)
4. iOS: `MFMessageComposeViewController` 호출
5. Android: `ACTION_SENDTO` Intent (`smsto:` URI)

### "QR 코드 보기"
- 풀스크린 모달 표시
- QR 코드 라이브러리: `react-native-qrcode-svg`
- 내용: 초대 URL (`https://chimeme.app/invite/{inviteId}`)
- 모달 내 "공유" 버튼 → 시스템 share sheet (QR 이미지 또는 URL)
- 모달 내 "닫기" → 모달만 닫고 그룹 생성 화면 유지

### "링크 복사"
- `expo-clipboard`로 URL 클립보드 복사
- 토스트 표시: "초대 링크를 복사했어요"
- 햅틱 피드백 (light impact)

### "카톡 공유" (P2)
- 베타에서는 비활성 (회색 + "곧 지원 예정" 툴팁)
- 향후 Kakao SDK 연동

## 초대 링크 처리 (외부 진입)

### 미설치 사용자
- 딥링크 → 앱스토어/Play Store로 리다이렉트 (Branch.io 또는 Firebase Dynamic Links 대안)
- 설치 후 첫 실행 시 deferred deep link로 invite ID 복원 → 인증 후 자동 가입

### 설치 사용자 (미로그인)
- 딥링크 → 앱 실행 → `auth-phone` (invite ID 메모리 보관) → 인증 → 자동 가입 → 그룹 상세

### 설치 사용자 (로그인)
- 딥링크 → 앱 실행 → 가입 확인 다이얼로그: "**[그룹명]**에 가입하시겠어요?"
- 확인 → `memberships` 생성 + FCM 토픽 구독 → 그룹 상세
- 취소 → 메인으로

### 만료된 링크
- 다이얼로그: "이 초대 링크는 만료됐어요. 그룹 멤버에게 새 링크를 받아주세요"

### 멤버 가득 찬 그룹 (8명)
- 다이얼로그: "이 그룹은 멤버가 가득 찼어요"
- 외부 진입의 경우 메인으로 이동

### 차단된 사용자 (강퇴 후 재초대 차단)
- `blockedFromGroups/{uid}_{groupId}` 존재 시 가입 차단
- 다이얼로그: "이 그룹에 가입할 수 없어요"

## 그룹 이름 검증 서버 측 (Cloud Function)
- 부적절 단어 사전 검사
- 길이/포맷 검증
- 위반 시 자동 마스킹 또는 그룹 자동 삭제

## 에러 처리
- 그룹 생성 트랜잭션 실패 → "잠시 후 다시 시도해 주세요" + 모달 유지
- 초대 링크 발급 실패 → 그룹은 생성되지만 링크 영역에 "다시 생성" 버튼

## 접근성
- 시트 진입 시 a11y focus를 그룹 이름 입력으로
- 모달 닫기 a11y: "그룹 만들기 취소"
- QR 모달 진입 시 announce: "QR 코드 화면, 닫으려면 화면 우상단 X 버튼 탭"

## 트래킹
- `group_create_view`
- `group_create_submit`
- `group_invite_link_copied` / `_sms_sent` / `_qr_viewed`
- `group_invite_redeemed` (외부 진입자가 가입 성공)
- `group_invite_expired` / `_full` / `_blocked`
