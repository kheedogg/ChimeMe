# auth-phone — 전화번호 입력

## 화면 키
`auth-phone`

## 진입로
- 미인증 상태로 앱 진입 시 첫 화면

## 목적
- 사용자 전화번호를 받아 Firebase Phone Auth SMS 코드 발송 트리거

## 사용자 시나리오
1. 국가 코드 + 전화번호 입력
2. "코드 받기" 탭 → SMS 발송 후 `auth-verify`로 이동

## 미정 / 디자인 TODO
- 국가 코드 selector UX (기본: 한국 +82)
- 약관 동의 체크박스 (이용약관 / 개인정보처리방침)
- 베타 화이트리스트 정책 (목록 외 번호 차단 여부)

## 후속 작성 필요
- `ui.md` — 입력 폼 / 키패드 / CTA
- `interactions.md` — 검증 규칙, 에러 케이스, 재전송 쿨다운
- `data.md` — Firebase Auth `signInWithPhoneNumber` 호출
