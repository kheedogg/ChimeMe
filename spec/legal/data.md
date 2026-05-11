# legal — Data

## 입력
- `type: 'tos' | 'privacy' | 'marketing'`
- 현재 사용자 uid (옵션 — 동의 기록 갱신용)

## 호스팅 / 저장

### 외부 정적 페이지 (옵션 A 권장)
- Firebase Hosting:
  - `https://chimeme.app/legal/tos/v1.html`
  - `https://chimeme.app/legal/privacy/v1.html`
  - `https://chimeme.app/legal/marketing/v1.html`
- HTML + CSS로 작성 (모바일 친화 디자인)
- 약관 버전마다 신규 파일 생성 (구버전 영구 유지 — 분쟁 대응)

### 버전 메타데이터 (Firebase Remote Config 또는 정적 JSON)
```json
{
  "legal": {
    "tos": {
      "currentVersion": "v1",
      "effectiveDate": "2026-05-12",
      "url": "https://chimeme.app/legal/tos/v1.html",
      "changeSummary": ""
    },
    "privacy": {
      "currentVersion": "v1",
      "effectiveDate": "2026-05-12",
      "url": "https://chimeme.app/legal/privacy/v1.html",
      "changeSummary": ""
    },
    "marketing": {
      "currentVersion": "v1",
      "effectiveDate": "2026-05-12",
      "url": "https://chimeme.app/legal/marketing/v1.html",
      "changeSummary": ""
    }
  }
}
```

## 동의 기록 (Firestore)

### `users/{uid}.agreements`
```typescript
type UserAgreements = {
  tosVersion: string;       // 'v1', 'v2', ...
  privacyVersion: string;
  marketingVersion?: string; // 마케팅 동의 시에만
  marketing: boolean;        // 마케팅 동의 여부
  agreedAt: Timestamp;       // 최초 동의 시각
  lastUpdatedAt: Timestamp;  // 재동의 시 갱신
};
```

### 갱신 (재동의 시)
```javascript
await updateDoc(doc(db, 'users', uid), {
  'agreements.tosVersion': newTosVersion,
  'agreements.privacyVersion': newPrivacyVersion,
  'agreements.lastUpdatedAt': serverTimestamp(),
});
```

## 약관 본문 가져오기

### 옵션 A: 외부 브라우저
```javascript
import * as WebBrowser from 'expo-web-browser';
import remoteConfig from '@react-native-firebase/remote-config';

async function openLegalDoc(type: 'tos' | 'privacy' | 'marketing') {
  const config = JSON.parse(remoteConfig().getString('legal'));
  const url = config[type].url;
  await WebBrowser.openBrowserAsync(url);
}
```

### 옵션 C: 인앱 마크다운 (오프라인 fallback)
- 앱 번들에 약관 마크다운 파일 포함:
  ```
  assets/legal/
    tos-v1.md
    privacy-v1.md
    marketing-v1.md
  ```
- React Native Asset으로 import 후 `react-native-markdown-display` 렌더링
- OTA 업데이트로 부분 갱신 가능

## 버전 체크 흐름

### 앱 시작 시 (Firebase Remote Config)
```javascript
import remoteConfig from '@react-native-firebase/remote-config';

await remoteConfig().fetchAndActivate();
const legalConfig = JSON.parse(remoteConfig().getString('legal'));

const userSnap = await getDoc(doc(db, 'users', user.uid));
const userAgreements = userSnap.data().agreements;

const tosNeedsUpdate = userAgreements.tosVersion !== legalConfig.tos.currentVersion;
const privacyNeedsUpdate = userAgreements.privacyVersion !== legalConfig.privacy.currentVersion;

if (tosNeedsUpdate || privacyNeedsUpdate) {
  // 재동의 모달 표시
  showLegalUpdateModal({
    tos: tosNeedsUpdate ? legalConfig.tos : null,
    privacy: privacyNeedsUpdate ? legalConfig.privacy : null,
  });
}
```

### 재동의 모달 응답 처리
```javascript
async function onAgreed(newVersions) {
  await updateDoc(doc(db, 'users', user.uid), {
    'agreements.tosVersion': newVersions.tos,
    'agreements.privacyVersion': newVersions.privacy,
    'agreements.lastUpdatedAt': serverTimestamp(),
  });
  closeLegalUpdateModal();
}

async function onDeclined() {
  navigate('account-delete');
}
```

## Security Rule

```
match /users/{uid} {
  allow update: if request.auth.uid == uid
    && request.resource.data.agreements.tosVersion is string
    && request.resource.data.agreements.privacyVersion is string;
}
```

## 인덱스
- `users.agreements.tosVersion` (개별 사용자 조회는 자동)
- 분석용: 버전별 동의 사용자 수 집계는 Firestore export → BigQuery 권장

## 약관 발행 가이드 (운영자용)

### 새 버전 배포 흐름
1. 새 약관 HTML 작성 (`v2.html`)
2. Firebase Hosting에 배포
3. Firebase Remote Config 업데이트:
   ```json
   "tos.currentVersion": "v2",
   "tos.url": "https://chimeme.app/legal/tos/v2.html",
   "tos.effectiveDate": "2026-08-01",
   "tos.changeSummary": "콘텐츠 보존 기간 명시, 신고 처리 절차 추가"
   ```
4. 사용자 앱 시작 시 재동의 모달 자동 표시
5. 30일 후 미동의 사용자 자동 비활성화 (P2)

### 시행일 처리
- `effectiveDate`가 미래면 재동의 모달은 시행일 이후에 표시
- 단, 신규 가입자는 항상 최신 버전에 동의

## 약관 백업 / 영구 보존
- 모든 약관 버전 파일은 영구 보존 (사용자 동의 시점의 약관 추적 가능)
- 분쟁 발생 시 `users/{uid}.agreements.tosVersion`으로 어떤 버전에 동의했는지 확인

## 분석 / 로깅
- 약관 화면 진입률
- 외부 링크 클릭 빈도
- 재동의 모달 표시율
- 재동의 동의 vs 거부 비율 (탈퇴 트리거 빈도)
- 버전별 사용자 분포

## 보안
- 약관 동의 정보는 본인만 갱신 가능 (Security Rule)
- 약관 호스팅 페이지는 HTTPS 강제
- Remote Config은 캐시 1시간 (정책 변경 즉시 반영 vs 비용 균형)
- 약관 변경 이력은 git으로 관리 + Firebase Hosting 영구 보존
