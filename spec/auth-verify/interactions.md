# auth-verify — Interactions

## 진입
- params: `verificationId`, `phoneNumber`, `sentAt`
- 진입 즉시 OTP 입력 박스에 포커스 → 키보드 자동 노출 + iOS/Android SMS auto-fill 대기

## 자동 채움
- iOS: `textContentType="oneTimeCode"` → 시스템이 SMS에서 코드 추출하여 키보드 위 제안
- Android: Google SMS Retriever API (`autoFillHints`) → 코드 자동 입력 (앱 해시 등록 필요)

## 코드 입력 흐름
1. 사용자 입력 또는 자동 채움으로 6자리 완성
2. 자동으로 검증 트리거 (Firebase `signInWithCredential(PhoneAuthProvider.credential(verificationId, code))`)
3. 검증 중 스피너 오버레이
4. 결과 분기:
   - 성공 + 기존 사용자 (`users/{uid}` 문서 존재) → `main`으로 이동
   - 성공 + 신규 사용자 → `onboarding`으로 이동
   - 실패 (`auth/invalid-verification-code`) → 박스 빨강 깜빡임 + "인증번호가 올바르지 않아요" + 박스 클리어
   - 실패 (`auth/code-expired`) → "코드가 만료됐어요. 재전송해 주세요" + 박스 회색 처리
   - 실패 (네트워크) → "연결을 확인해 주세요" 토스트 + 박스 유지

## 코드 만료
- 5분(300초) 카운트다운 시각화
- 만료 시점: 박스 회색 + 자동 검증 비활성 + "재전송" 강조

## 재전송 쿨다운
- 코드 발송 후 **60초 쿨다운**
- 쿨다운 중: 재전송 텍스트 회색 + 카운트다운 표시
- 쿨다운 종료: "재전송" 탭 가능 → 탭 시:
  1. `auth-phone`과 동일한 로직으로 새 코드 발송
  2. 카운트다운 5분 리셋, 박스 클리어, 포커스 이동
  3. Rate limit 정책 (D-12-12) 적용 — 한도 초과 시 차단

## 시도 횟수 제한
- 동일 `verificationId`로 검증 실패 횟수: **10회**
- 초과 시: 박스 잠금 + "잠시 후 다시 시도해 주세요" 안내 → "전화번호 다시 입력" 링크로 `auth-phone` 복귀

## 뒤로가기
- ← 버튼 또는 시스템 뒤로가기 → `auth-phone`으로 복귀
- 복귀 시 입력했던 전화번호는 자동 채움 (`AsyncStorage.auth.lastPhoneNumber`)
- 약관 동의 상태도 보존 (`AsyncStorage.auth.agreements.draft`)

## 신규 사용자 판정 (성공 후)
- Firebase Auth 성공 시 `user.metadata.creationTime === user.metadata.lastSignInTime` 비교
- 또는 더 안전하게: Firestore `users/{uid}` 문서 조회 → 없으면 신규 → `onboarding`
- `onboarding` 진입 시 약관 동의 draft를 함께 전달

## 에러 / 빈 상태
- 코드 잘못 입력 시 박스 빨강 0.5초 깜빡임 + haptic (light impact)
- 만료 시 박스 회색 + 텍스트 안내
- 네트워크 오류 시 토스트만 표시, 박스 상태 유지

## 보안
- `verificationId`는 메모리에만 보관, 화면 이탈 시 클리어
- 검증 성공 직후 Firebase ID 토큰을 secure storage(`expo-secure-store`)에 저장
- 6자리 코드는 입력 후 즉시 사용, 별도 저장 금지

## 접근성
- 박스 1자리 입력될 때마다 a11y announce: "1자리 입력됨"
- 자동 채움 트리거 시: "인증번호 자동 입력됨, 검증 중"
- 검증 결과 a11y live region으로 즉시 안내

## 트래킹 이벤트
- `auth_verify_view`
- `auth_verify_attempt` (success/failure 분기)
- `auth_verify_resend` (재전송 트리거)
- `auth_verify_expired` (만료 도달)
