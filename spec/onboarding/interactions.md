# onboarding — Interactions

## 진입
- `auth-verify` 성공 직후 신규 사용자(`users/{uid}` 부재)일 때만
- params: `agreements.draft` (auth-phone에서 동의한 약관 정보)
- 기존 사용자는 자동으로 `main` 진입 (이 화면 거치지 않음)

## 스텝 1: 생년월일

### 검증
- 만 14세 계산 기준: 오늘 - 입력 생년월일 ≥ 14년
- 정확한 만 나이: 윤년/생일 도래 여부 고려 (`date-fns` `differenceInYears`)
- 미래 날짜 입력 차단 (Picker `maxDate` 제약)

### "다음" 탭
1. 만 14세 이상 검증
2. 메모리에 `birthDate` 보관
3. 스텝 2로 전환 (좌측 슬라이드)

### 만 14세 미만 시
- CTA 비활성 상태로 유지
- 인라인 안내 표시
- "다른 번호로 가입하시려면 [전화번호 다시 입력]" — `auth-phone`으로 복귀 옵션 제공 (현 계정 미생성 상태로 로그아웃)

## 스텝 2: 닉네임 + 프로필 사진

### 닉네임 검증
| 입력 | 동작 |
|---|---|
| 빈 값 | CTA 비활성 |
| 1~20자 | CTA 활성 |
| 21자+ | 입력 차단 |
| 부적절 단어 (욕설) | 인라인 에러 "사용할 수 없는 닉네임" |
| 앞/뒤 공백 | 자동 trim |
| 이모지 | 허용 |

부적절 단어 사전: 클라이언트 측 기본 필터링 (한국어 욕설 사전) + 서버 추가 검증 (Cloud Function).

### 프로필 사진 선택
- ActionSheet 표시: "카메라로 촬영" / "갤러리에서 선택" / "기본 이미지 사용"
- 권한 요청: `expo-image-picker` 카메라/갤러리 권한
- 권한 거부 시: "사진 권한이 필요합니다. 설정에서 허용해 주세요" + "기본 이미지 사용" 옵션 fallback
- 선택 후 크롭: 정사각형 크롭 강제 (`expo-image-picker` `allowsEditing: true, aspect: [1, 1]`)
- 즉시 업로드 시작 → 아바타에 스피너 표시 → 완료 시 미리보기

### "ChimeMe 시작하기" 탭
1. CTA 비활성 + 스피너
2. (사진 미업로드 상태면) 사진 업로드 완료 대기
3. Firestore 트랜잭션으로 `users/{uid}` 문서 생성 (data.md 참조)
4. FCM 토큰 발급 + `users/{uid}.fcmTokens[]`에 등록
5. `main` 화면으로 replace 라우팅 (뒤로가기 불가)

## 뒤로가기 차단
- 스텝 1/2 모두 시스템 뒤로가기 차단
- 이탈 시 작성 중인 정보(`birthDate`, `nickname`, `photoUri`)는 메모리 유지
- 단, 만 14세 미만 차단 화면에서만 "전화번호 다시 입력" 경로 허용

## 앱 강제 종료 → 재진입
- Firebase Auth는 이미 인증 상태 → `main`으로 가려 하지만 `users/{uid}` 문서 부재 감지 → onboarding 재진입
- AsyncStorage에 `onboarding.draft = { birthDate?, nickname?, photoUri? }` 저장 → 마지막 스텝부터 재개

## 에러 처리
- 사진 업로드 실패 → 토스트 + "기본 이미지로 시작" 폴백 옵션 제공
- Firestore write 실패 → "잠시 후 다시 시도해 주세요" + CTA 활성 복귀
- 만 14세 검증 우회 시도 (생년월일 직접 조작) → 서버 측 Cloud Function에서 재검증 → 부적합 시 계정 자동 삭제

## 접근성
- 스텝 전환 시 a11y announce: "2단계 중 2단계, 프로필 설정"
- 이미지 업로드 진행 상태 a11y live region
- 부적절 닉네임 에러는 input 옆 인라인 + a11y assertive

## 트래킹
- `onboarding_step1_view` / `onboarding_step1_submit`
- `onboarding_step1_underage` (만 14세 미만 차단)
- `onboarding_step2_view` / `onboarding_step2_submit`
- `onboarding_photo_uploaded` / `onboarding_photo_skipped`
- `onboarding_complete`
