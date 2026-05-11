# onboarding — Data

## 입력
- Firebase Auth `currentUser` (uid, phoneNumber)
- `agreements.draft: { tos: bool, privacy: bool, marketing: bool }` — 이전 화면에서 전달
- 메모리/AsyncStorage:
  - `onboarding.draft.birthDate: string` (YYYY-MM-DD)
  - `onboarding.draft.nickname: string`
  - `onboarding.draft.photoUri: string` (로컬 URI)

## 생성 데이터

### Firestore: `users/{uid}` 문서 생성 (트랜잭션)
```javascript
await runTransaction(db, async (tx) => {
  const userRef = doc(db, 'users', user.uid);
  const userSnap = await tx.get(userRef);
  if (userSnap.exists()) {
    throw new Error('USER_ALREADY_EXISTS'); // race condition 방지
  }
  tx.set(userRef, {
    uid: user.uid,
    phoneNumber: user.phoneNumber,
    displayName: nickname,
    photoUrl: uploadedPhotoUrl ?? null,
    birthDate: birthDate, // 'YYYY-MM-DD'
    timezone: 'Asia/Seoul', // D-12-01: 사용자 기본 timezone
    agreements: {
      tosVersion: 'v1',
      privacyVersion: 'v1',
      marketing: agreements.marketing,
      agreedAt: serverTimestamp(),
    },
    notifications: {
      enabled: true, // Q-12-09: 기본 ON
      digestMode: false,
      quietHoursStart: '23:00',
      quietHoursEnd: '07:00',
    },
    fcmTokens: [],
    createdAt: serverTimestamp(),
    updatedAt: serverTimestamp(),
  });
});
```

### Storage: 프로필 사진 업로드
- 경로: `profiles/{uid}.jpg`
- 포맷: JPEG 80% quality
- 해상도: 400×400 (정사각형 크롭 후 리사이즈)
- 업로드 후 다운로드 URL 획득하여 `users/{uid}.photoUrl`에 저장

### FCM 토큰 등록
```javascript
const fcmToken = await messaging().getToken();
await updateDoc(userRef, {
  fcmTokens: arrayUnion(fcmToken),
});
```

## 서버 측 검증 (Cloud Function)
사용자 생성 직후 트리거:
```javascript
exports.onUserCreated = onDocumentCreated('users/{uid}', async (event) => {
  const data = event.data.data();
  
  // 만 14세 검증 (Q-12-03)
  const age = differenceInYears(new Date(), parseISO(data.birthDate));
  if (age < 14) {
    // 우회 시도 감지 → 계정 자동 삭제
    await admin.auth().deleteUser(event.params.uid);
    await event.data.ref.delete();
    console.warn(`Underage user deleted: ${event.params.uid}`);
    return;
  }
  
  // 약관 동의 검증
  if (!data.agreements?.tosVersion || !data.agreements?.privacyVersion) {
    console.error(`Missing agreements: ${event.params.uid}`);
  }
  
  // 닉네임 부적절 단어 서버 측 검증
  if (containsProhibitedWords(data.displayName)) {
    await event.data.ref.update({ displayName: `사용자${Math.random().toString(36).slice(2, 8)}` });
  }
});
```

## 캐시 / 로컬 저장
- 입력 중 데이터는 AsyncStorage `onboarding.draft`에 단계별 저장
- 완료 후 draft 삭제

## 인덱스
- `users` 컬렉션: `uid` (자동)
- `users` 컬렉션: `phoneNumber` 단일 필드 인덱스 (전화번호 중복 검색용)

## Security Rule
```
match /users/{uid} {
  allow read: if request.auth.uid == uid;
  allow create: if request.auth.uid == uid
    && request.resource.data.uid == uid
    && request.resource.data.phoneNumber == request.auth.token.phone_number
    && request.resource.data.displayName.size() >= 1
    && request.resource.data.displayName.size() <= 20
    && request.resource.data.agreements.tosVersion != null
    && request.resource.data.agreements.privacyVersion != null;
  allow update: if request.auth.uid == uid;
  allow delete: if false; // 계정 삭제는 Cloud Function 경유
}

match /profiles/{uid} {
  allow read: if request.auth != null;
  allow write: if request.auth.uid == uid
    && request.resource.size < 1 * 1024 * 1024 // 1MB 제한
    && request.resource.contentType.matches('image/.*');
}
```

## 분석 / 로깅
- 온보딩 완료율 (스텝 1 진입 → 스텝 2 완료)
- 평균 소요 시간
- 사진 업로드 비율 (vs 건너뛴 비율)
- 만 14세 미만 차단율
- 부적절 닉네임 차단율

## 보안
- 프로필 사진 메타데이터 EXIF 제거 (위치 정보 등)
- 클라이언트 검증 후 서버 검증 이중화 (생년월일, 닉네임, 약관)
