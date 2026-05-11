# legal — 이용약관 / 개인정보처리방침

## 화면 키
`legal`

## 진입로
- `auth-phone` 약관 체크박스 옆 "보기 >"
- `profile` → "이용약관" / "개인정보처리방침"
- 약관 변경 시 앱 시작 시 재동의 모달

## 목적
- 이용약관(ToS) / 개인정보처리방침(Privacy Policy) 전문 표시
- 약관 동의 기록 추적
- Apple / Google 앱 심사 요건 충족

## 사용자 시나리오
1. 약관 진입로 탭 → `legal/{type}` 화면 진입
2. 약관 전문 스크롤 열람
3. 외부 링크가 있으면 별도 브라우저로 이동
4. ← 뒤로가기로 이전 화면 복귀

## 약관 종류 (라우트별)
| 라우트 | 내용 | 동의 의무 |
|---|---|---|
| `legal/tos` | 이용약관 | **필수** |
| `legal/privacy` | 개인정보처리방침 | **필수** |
| `legal/marketing` | 마케팅 정보 수신 동의 | 선택 |

## 표시 방식 결정 (호스팅 전략)

### 옵션 A: 외부 정적 페이지 (권장)
- Firebase Hosting에 `https://chimeme.app/legal/tos.html` 등으로 호스팅
- 앱에서 `expo-web-browser`로 외부 브라우저 열기
- 장점: 약관 변경 시 앱 업데이트 불필요, SEO 친화, 법무 검토 후 즉시 반영
- 단점: 인터넷 연결 필요

### 옵션 B: 인앱 WebView
- `expo-web-browser` 인앱 브라우저로 표시
- 장점: 외부 이탈 없음, UI 통일성
- 단점: 약관 변경 시 동일하게 외부 호스팅 의존

### 옵션 C: 인앱 마크다운 렌더링
- 앱 번들에 마크다운 포함, `react-native-markdown-display` 렌더링
- 장점: 오프라인 가능
- 단점: 약관 변경 시 앱 업데이트 필요 (OTA로 부분 해결)

**권장: 옵션 A (외부 정적 페이지) + 옵션 C (오프라인 fallback)**

## 약관 버전 관리
- 약관 텍스트는 버전 번호로 관리:
  - `legal/tos/v1.html`, `legal/tos/v2.html`
  - `legal/privacy/v1.html`, `legal/privacy/v2.html`
- `users/{uid}.agreements.tosVersion / privacyVersion`에 동의한 버전 기록
- 새 버전 배포 시 앱 시작 시 버전 비교 → 다르면 재동의 모달

## 약관 변경 시 재동의 모달
- 앱 시작 시 (또는 백그라운드 → 포그라운드 복귀 시) 체크
- 변경된 항목만 강조 표시 ("개인정보처리방침이 업데이트되었습니다")
- 사용자가 동의해야 앱 사용 가능
- 거부 시 앱 종료 또는 계정 삭제 안내

## 미정 / 향후
- 약관 작성 (법무 검토 필요) — 사용자가 직접 진행해야 할 영역
- 약관 호스팅 URL 확정 (Firebase Hosting vs 별도 도메인)
- 약관 다국어 지원 (베타는 한국어만)

## 후속 작성
- `ui.md` ✓ `interactions.md` ✓ `data.md` ✓
