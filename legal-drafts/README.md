# legal-drafts/ — 법무 문서 초안

> ⚠️ **본 디렉토리의 모든 문서는 Claude AI 작성 초안입니다.**
> 출시 전 반드시 **변호사 / 법무팀 / 개인정보보호 책임자 검토**가 필요합니다.

## 파일 목록

| 파일 | 용도 | 검토 필요 |
|---|---|---|
| `tos-v1.md` | 이용약관 초안 v1 | ✅ 변호사 |
| `privacy-v1.md` | 개인정보처리방침 초안 v1 | ✅ 변호사 + DPO |

## 작성 기준

- 한국 출시 우선: KISA, 방통위, 개인정보보호위원회 가이드라인 준수
- 글로벌 확장 대비: GDPR Article 17, 20 일부 반영
- ChimeMe의 확정 정책 반영:
  - D-12-10: 영상 7일 보존
  - Q-12-03: 만 14세 이상 가입
  - Q-12-04: 신고/차단 정책
  - Q-12-08: 채팅 90일 보존
  - D-12-09: 계정 삭제 시 데이터 처리

## 검토 / 배포 흐름

### 1. Placeholder 채우기
다음 항목들을 실제 값으로 교체:
- `[CHIMEME_ENTITY_NAME]` — 회사명 / 사업자명
- `[CEO_NAME]` — 대표자 성명
- `[COMPANY_ADDRESS]` — 사업장 소재지
- `[BUSINESS_NUMBER]` — 사업자등록번호
- `[DPO_NAME]` — 개인정보보호 책임자 성명
- `[DPO_PHONE]` — 책임자 연락처
- `2026-XX-XX` — 시행일

### 2. 법무 검토
- 변호사 검토 (이용약관 + 개인정보처리방침)
- 개인정보보호 책임자(DPO) 지정 및 동의

### 3. HTML 변환 + 호스팅
```bash
# 마크다운 → HTML 변환 (예: marked, pandoc)
npx marked legal-drafts/tos-v1.md > public/legal/tos/v1.html
npx marked legal-drafts/privacy-v1.md > public/legal/privacy/v1.html

# Firebase Hosting 배포
firebase deploy --only hosting
```

배포 후 URL:
- `https://chimeme.app/legal/tos/v1.html`
- `https://chimeme.app/legal/privacy/v1.html`

### 4. Remote Config 등록
Firebase Console → Remote Config → 매개변수 추가 (`legal`):
```json
{
  "tos": {
    "currentVersion": "v1",
    "effectiveDate": "2026-XX-XX",
    "url": "https://chimeme.app/legal/tos/v1.html",
    "changeSummary": ""
  },
  "privacy": {
    "currentVersion": "v1",
    "effectiveDate": "2026-XX-XX",
    "url": "https://chimeme.app/legal/privacy/v1.html",
    "changeSummary": ""
  }
}
```

## 버전 관리

- 약관 개정 시 새 버전 파일 추가 (`tos-v2.md`, `privacy-v2.md`)
- 구버전은 영구 보존 (`tos-v1.md` 유지)
- `spec/legal/data.md` 참조: 사용자 동의 버전 추적
- 약관 변경 시 시행 30일 전 사전 공지 + 재동의 모달 (불리한 변경 시 30일, 그 외 7일)

## 주의 사항

1. **본 초안은 법적 효력이 없습니다.** 출시 전 반드시 법무 검토.
2. **위탁 업체 목록**(개인정보처리방침 제5조)은 실제 사용하는 서비스에 맞춰 갱신.
3. **국외 이전 동의**(제5.1조)는 한국 개인정보보호법 시행령에 따라 별도 동의 절차 필요할 수 있음.
4. **미성년자 보호 조항**은 KISA 가이드라인 및 청소년 보호법 추가 검토.
5. **회사 정보**가 확정되지 않으면 출시 불가 (Apple/Google 심사 요건).

## 참고 자료

- [한국 개인정보보호법](https://www.law.go.kr/법령/개인정보보호법)
- [KISA 개인정보보호 가이드라인](https://privacy.kisa.or.kr)
- [정보통신망법](https://www.law.go.kr/법령/정보통신망이용촉진및정보보호등에관한법률)
- [GDPR (영문)](https://gdpr.eu/)
