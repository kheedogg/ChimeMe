# auth-phone — Interactions

## 진입
- 인증 토큰 부재 시 첫 화면. 인증 토큰 존재 시 자동으로 `main`으로 라우팅 (이 화면 진입 차단).
- 백버튼(Android) / 스와이프(iOS): 비활성 (앱 종료 외 경로 없음)

## 입력 검증

### 전화번호
| 입력 | 동작 |
|---|---|
| 숫자 외 문자 | 무시 (입력 자체 차단) |
| 0으로 시작 안 함 | 인라인 에러 "전화번호는 010으로 시작합니다" |
| 길이 < 11 | CTA 비활성, 에러는 표시 안 함 (입력 중) |
| 길이 11, 형식 OK | CTA 활성 |
| 길이 > 11 | 입력 자체 차단 |

### 약관
- 필수 2개 모두 ON 필요. 하나라도 OFF면 CTA 비활성.
- 선택 동의는 `users/{uid}.agreements.marketing`에 boolean 저장.
- "모두 동의" 체크 시 모두 ON, 해제 시 모두 OFF.

## CTA "코드 받기" 탭
1. CTA 비활성화 + 스피너 표시
2. App Check 토큰 발급 (D-12-04)
3. Firebase `signInWithPhoneNumber(+82{phone}, recaptchaVerifier)` 호출
4. 결과 분기:
   - 성공 → `auth-verify`로 이동 (`verificationId` + `phoneNumber` + 발송 시각 전달)
   - 실패 (`auth/invalid-phone-number`) → 인라인 에러 "전화번호 형식이 올바르지 않습니다"
   - 실패 (`auth/too-many-requests`) → "잠시 후 다시 시도해 주세요" 토스트 + 60초 쿨다운
   - 실패 (App Check fail) → "보안 검증에 실패했습니다. 앱을 재시작해 주세요"
   - 실패 (네트워크) → "연결을 확인해 주세요" 토스트

## Rate Limit (D-12-12)
- 클라이언트 측: 마지막 발송 시각으로부터 60초 미만이면 즉시 차단 + 카운트다운 표시
- 서버 측 (Cloud Function): 동일 IP 1시간 5건 / 동일 번호 1일 10건 초과 시 24시간 차단

## 약관 보기
- "보기 >" 탭 → 외부 브라우저(`expo-web-browser`)로 약관 URL 열기 (인앱 모달도 가능)
- 약관 URL은 `spec/legal` 화면에서 호스팅 또는 외부 정적 페이지

## 에러 / 빈 상태
- 네트워크 오프라인 → 화면 상단 배너 "오프라인 상태입니다" + CTA 비활성
- App Check 실패 (루팅/탈옥 기기 의심) → "기기 환경에서 ChimeMe를 사용할 수 없습니다" 풀스크린

## 키보드 처리
- 화면 진입 시 전화번호 필드 자동 포커스 → 키보드 자동 노출
- 약관 체크박스 영역은 키보드 위로 자동 스크롤 (KeyboardAvoidingView)

## 접근성
- VoiceOver/TalkBack로 약관 항목 읽기 가능
- 동적 폰트 크기 대응

## 트래킹 이벤트 (향후)
- `auth_phone_view`
- `auth_phone_terms_view` (어떤 약관 보기 클릭했는지)
- `auth_phone_submit_attempt` (성공/실패 분기)
