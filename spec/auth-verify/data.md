# auth-verify — Data

## 입력
- `verificationId: string` — 이전 화면(`auth-phone`)에서 전달
- `phoneNumber: string` — E.164 형식 (마스킹 표시용)
- `sentAt: ISO timestamp` — 카운트다운 기준
- (옵션) `agreements.draft: { tos, privacy, marketing }` — 신규 사용자 시 onboarding으로 전달

## 외부 호출

### 코드 검증
- `signInWithCredential(PhoneAuthProvider.credential(verificationId, code))`
- 출력: `UserCredential`
  - `user.uid`
  - `user.phoneNumber`
  - `user.metadata.creationTime`
  - `user.metadata.lastSignInTime`

### 신규 사용자 판정
```javascript
const docSnap = await getDoc(doc(db, 'users', user.uid));
const isNewUser = !docSnap.exists();
```

### 재전송
- `auth-phone`과 동일 — `signInWithPhoneNumber` 재호출, 새 `verificationId` 발급

## 생성 데이터
- 이 화면 자체에서는 Firestore 문서 생성 안 함
- 신규 사용자 분기 시 `onboarding`이 `users/{uid}` 생성 책임

## 토큰 관리
- 검증 성공 시:
  - Firebase ID 토큰 → `expo-secure-store`에 저장 (`auth.idToken`)
  - Refresh 토큰은 Firebase SDK가 자동 관리
  - FCM 토큰 발급 트리거 → `users/{uid}.fcmTokens[]`에 등록 (onboarding 또는 main 진입 시)

## 시도 횟수 카운트
- 메모리에 `attemptCount` 보관
- 10회 초과 시 잠금 + UI 잠금 상태로 전환
- 재전송 시 카운트 리셋

## Security Rule 메모
- 이 화면도 인증 완료 시점까지 Firestore 직접 접근 없음
- 검증 성공 후 `users/{uid}` 조회는 본인 문서만 가능 (`auth.uid == uid`)

## 분석 / 로깅
- 검증 시도 / 성공 / 실패 카운터
- 신규/기존 사용자 비율 추적
- 평균 입력 완료 시간 (자동 채움 vs 수동 입력)
- 재전송 빈도 (높으면 SMS 미수신 문제 의심)

## 인덱스
- `users` 컬렉션: `uid` (자동 인덱스)

## 보안
- 검증 성공 직후 모든 인증 상태를 secure storage로 이전
- 메모리상 `verificationId`, 입력한 코드 즉시 클리어
- 로그에 OTP 코드 절대 기록 금지 (필요 시 마스킹: `******`)
