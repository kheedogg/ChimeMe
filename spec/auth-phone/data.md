# auth-phone — Data

## 사용 데이터

### 외부 호출
- Firebase Auth `signInWithPhoneNumber(phoneNumber, recaptchaVerifier)`
  - 입력: E.164 형식 (예: `+821012345678`)
  - 출력: `verificationId` (다음 화면에 전달)
- Firebase App Check 토큰 (D-12-04)
  - iOS: DeviceCheck provider
  - Android: Play Integrity provider
  - 디버그 빌드: debug provider (`AppCheckDebugProvider`)

### 로컬 저장 (AsyncStorage)
- `auth.lastSentAt: ISO timestamp` — 60초 쿨다운 계산
- `auth.lastPhoneNumber: string` — 자동 채움 (선택)
- `auth.agreements.draft: { tos: bool, privacy: bool, marketing: bool }` — 화면 이탈 후 재진입 시 복원

## 생성 데이터
- 이 화면 자체에서는 Firestore 문서를 생성하지 않음
- 약관 동의 정보는 `users/{uid}` 문서 생성 시점(`onboarding`)에 함께 기록

## Firestore Security Rule 메모
- 이 화면은 인증 전 단계라 Firestore 직접 접근 없음
- App Check 토큰 검증은 Firebase Auth 단계에서 수행

## SMS 비용 보호 (D-12-12)
서버 측 Cloud Function `beforeSendSms`:
```javascript
exports.beforeSendSms = functions.auth.user().beforeCreate(async (user, context) => {
  const ip = context.rawRequest.ip;
  const phone = user.phoneNumber;
  
  // IP rate limit: 1시간 5건
  const ipKey = `rate_limit:ip:${ip}`;
  const ipCount = await incrementRateLimit(ipKey, 3600);
  if (ipCount > 5) throw new HttpsError('resource-exhausted', 'IP rate limit exceeded');
  
  // Phone rate limit: 1일 10건
  const phoneKey = `rate_limit:phone:${phone}`;
  const phoneCount = await incrementRateLimit(phoneKey, 86400);
  if (phoneCount > 10) throw new HttpsError('resource-exhausted', 'Phone rate limit exceeded');
});
```
- Redis 또는 Firestore TTL 컬렉션으로 카운터 구현
- 한도 초과 시 24시간 차단 + 관리자 알림 (Slack webhook)

## 약관 버전 관리
- 약관 텍스트는 `spec/legal/` 또는 외부 정적 호스팅 (Firebase Hosting)
- 버전은 `legal/{tos|privacy}/v1.json` 형식
- `users/{uid}.agreements.tosVersion / privacyVersion`에 동의한 버전 기록
- 약관 변경 시 앱 시작 시 버전 비교 → 재동의 모달

## 분석 / 로깅
- 발송 시도 / 성공 / 실패 별 카운터 (Firebase Analytics 이벤트)
- 실패 사유별 분류 로그 (rate limit / invalid phone / App Check / network)
- toll fraud 모니터링: Cloud Logging 알림 — 발송 실패율 > 30% 시 알림

## 인덱스
- 해당 없음 (Firestore 미사용)

## 보안 메모
- `verificationId`는 메모리에만 보관 (AsyncStorage 저장 금지)
- 화면 이탈 시 메모리에서 클리어
- 약관 동의 draft는 보관해도 무방 (민감 정보 아님)
