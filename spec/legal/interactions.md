# legal — Interactions

## 진입
- `auth-phone` 약관 체크박스 "보기 >" 탭
- `profile` 메뉴 "이용약관" 또는 "개인정보처리방침" 탭
- 약관 변경 감지 시 강제 모달 표시

## 표시 방식 (옵션 A 권장)

### 외부 브라우저 호출
```javascript
import * as WebBrowser from 'expo-web-browser';

await WebBrowser.openBrowserAsync('https://chimeme.app/legal/tos');
```
- 시스템 기본 브라우저 또는 SFSafariViewController (iOS) / Custom Tab (Android) 활용
- 사용자가 ← 또는 닫기로 앱 복귀

## 약관 버전 체크 (앱 시작 시)

### 시작 흐름
1. 앱 시작 시 `users/{uid}.agreements.{tosVersion, privacyVersion}` fetch
2. Firebase Remote Config 또는 정적 JSON에서 현재 버전 fetch:
   ```json
   {
     "tos": { "currentVersion": "v2", "effectiveDate": "2026-06-01" },
     "privacy": { "currentVersion": "v1", "effectiveDate": "2026-05-12" }
   }
   ```
3. 사용자 동의 버전 < 현재 버전이면 재동의 모달 표시

### 재동의 모달
1. 변경된 약관 종류 표시 (체크박스로 확인)
2. 각 약관 "내용 보기 >" → 전체 본문 표시 (옵션 B 인앱 또는 옵션 A 외부 브라우저)
3. 변경 요약 표시 (Remote Config에서 함께 fetch)
4. 사용자가 모든 변경 약관 체크 → "동의하고 계속하기" CTA 활성
5. 동의 탭:
   - `users/{uid}.agreements.{tosVersion, privacyVersion, agreedAt}` 갱신
   - 모달 닫기 → 정상 앱 사용
6. 거부 (계정 삭제 링크 탭):
   - `account-delete` 라우트로 이동
   - 또는 앱 종료

## 외부 링크 처리
- 약관 내 외부 URL 클릭 시:
  - "외부 브라우저로 이동합니다" 확인 다이얼로그 (선택 — 또는 즉시 이동)
  - `expo-web-browser` 또는 `Linking.openURL`로 처리

## 오프라인 모드
- 인앱 마크다운 렌더링(옵션 C)이라면 오프라인에서도 표시 가능
- 외부 브라우저(옵션 A)는 인터넷 필요 → 오프라인 시 "인터넷 연결 후 다시 시도해 주세요" 안내

## 약관 동의 흐름 (auth-phone에서)

### 체크박스 동의 상태 보관
- AsyncStorage에 임시 저장 (`auth.agreements.draft`)
- 인증 완료 시점에 `users/{uid}.agreements`에 영구 저장

### "보기 >" 탭
1. 외부 브라우저로 약관 페이지 열기
2. 사용자가 약관 읽고 앱 복귀
3. 체크박스 상태 유지

## 약관 검색 (옵션 B 인앱 표시 시)
- 우상단 돋보기 아이콘 → 검색 입력
- 매칭된 키워드 하이라이트 + 다음/이전 이동 버튼
- 향후 기능 (베타에는 미지원)

## 트래킹
- `legal_view` (type: tos / privacy / marketing)
- `legal_external_link_clicked` (도메인)
- `legal_update_modal_view`
- `legal_update_agreed`
- `legal_update_declined` (계정 삭제 흐름으로 이동)

## 에러 처리
- 외부 브라우저 열기 실패 → 인앱 fallback (옵션 C 마크다운)
- 약관 버전 체크 실패 → 기본 동작 유지 (다음 시작 시 재시도)
- 모든 약관 fetch 실패 → "약관을 불러올 수 없어요" 안내 + 재시도 버튼

## 접근성
- 외부 브라우저 이동 시 a11y announce
- 재동의 모달 a11y focus를 첫 체크박스로
- 변경 사항 a11y emphasize
- 거부 옵션 a11y: "계정 삭제 화면으로 이동, 주의 필요"
