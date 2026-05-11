# profile — Data

## 입력
- 현재 사용자 uid (Firebase Auth)

## 사용 데이터

### Firestore
- `users/{uid}` 본인 문서 read
  - `displayName`, `photoUrl`, `phoneNumber`, `birthDate`, `agreements`, `notifications`
- `memberships` where `uid == currentUser.uid` → 내 그룹 목록
- `groups` where `ownerId == currentUser.uid` → 내가 소유한 그룹
- `blocks` where `blockerId == currentUser.uid` → 차단 목록

### Storage
- `profiles/{uid}.jpg` — 프로필 사진

## 갱신 작업

### 닉네임 변경
```javascript
await updateDoc(doc(db, 'users', uid), {
  displayName: newNickname,
  updatedAt: serverTimestamp(),
});
```
- 그룹 멤버 표시는 `users/{uid}` 참조로 자동 반영 (별도 동기화 불필요)

### 사진 변경
```javascript
// 새 사진 업로드
const ref = storageRef(storage, `profiles/${uid}.jpg`);
await uploadBytes(ref, blob);
const photoUrl = await getDownloadURL(ref);
await updateDoc(doc(db, 'users', uid), { photoUrl });

// 사진 삭제
await deleteObject(storageRef(storage, `profiles/${uid}.jpg`));
await updateDoc(doc(db, 'users', uid), { photoUrl: null });
```

### 차단 해제
```javascript
await deleteDoc(doc(db, 'blocks', `${myUid}_${theirUid}`));
```

## 로그아웃
```javascript
// FCM 토큰 정리
const currentToken = await messaging().getToken();
await updateDoc(doc(db, 'users', uid), {
  fcmTokens: arrayRemove(currentToken),
});

// 그룹 토픽 구독 해제
const myMembershipsSnap = await getDocs(query(
  collection(db, 'memberships'),
  where('uid', '==', uid)
));
for (const m of myMembershipsSnap.docs) {
  await messaging().unsubscribeFromTopic(`group_${m.data().groupId}`);
}

// Firebase 로그아웃
await signOut(auth);

// 로컬 데이터 클리어
await AsyncStorage.multiRemove(['auth.idToken', 'upload_queue.v1']);
await SecureStore.deleteItemAsync('auth.session');
```

## 계정 삭제 Cloud Function (D-12-09)

```javascript
exports.deleteAccount = onCall(async (request) => {
  const uid = request.auth?.uid;
  if (!uid) throw new HttpsError('unauthenticated', 'Auth required');
  
  const { ownerGroupDecisions } = request.data;
  // ownerGroupDecisions: [{ groupId, action: 'dissolve' | 'transfer', newOwnerId? }]
  
  // 1. 소유 그룹 처리
  for (const decision of ownerGroupDecisions) {
    if (decision.action === 'dissolve') {
      await dissolveGroup(decision.groupId);
    } else if (decision.action === 'transfer') {
      await transferOwnership(decision.groupId, decision.newOwnerId);
    }
  }
  
  // 2. 본인이 멤버인 그룹에서 본인 제거
  const myMembershipsSnap = await getDocs(query(
    collection(db, 'memberships'),
    where('uid', '==', uid)
  ));
  
  for (const m of myMembershipsSnap.docs) {
    const gid = m.data().groupId;
    
    // 그룹에서 본인 제거
    await updateDoc(doc(db, 'groups', gid), {
      memberIds: arrayRemove(uid),
    });
    
    // 본인 영상 Storage 삭제
    const myPostsSnap = await getDocs(query(
      collection(db, 'groups', gid, 'posts'),
      where('authorId', '==', uid)
    ));
    for (const p of myPostsSnap.docs) {
      const path = p.data().videoPath;
      try {
        await deleteObject(storageRef(storage, path));
      } catch (e) { /* ignore not found */ }
      await deleteDoc(p.ref);
    }
    
    // 본인 채팅 메시지 익명화 (메시지 자체는 보존)
    const myMessagesSnap = await getDocs(query(
      collection(db, 'groups', gid, 'messages'),
      where('authorId', '==', uid)
    ));
    const anonymousId = `deleted_${hashUid(uid)}`;
    const batch = writeBatch(db);
    myMessagesSnap.docs.forEach(d => {
      batch.update(d.ref, { authorId: anonymousId });
    });
    await batch.commit();
    
    // memberships 삭제
    await deleteDoc(m.ref);
    
    // FCM 토픽 구독 해제
    // (이미 클라이언트에서 처리했지만 서버 측에서도 확인)
  }
  
  // 3. blocks 정리 (양방향)
  const myBlocksSnap = await getDocs(query(
    collection(db, 'blocks'),
    where('blockerId', '==', uid)
  ));
  myBlocksSnap.forEach(d => d.ref.delete());
  
  const blockedMeSnap = await getDocs(query(
    collection(db, 'blocks'),
    where('blockedId', '==', uid)
  ));
  blockedMeSnap.forEach(d => d.ref.delete());
  
  // 4. 본인 프로필 사진 Storage 삭제
  try {
    await deleteObject(storageRef(storage, `profiles/${uid}.jpg`));
  } catch (e) { /* ignore */ }
  
  // 5. users/{uid} 문서 삭제
  await deleteDoc(doc(db, 'users', uid));
  
  // 6. Firebase Auth 계정 삭제
  await admin.auth().deleteUser(uid);
  
  return { success: true };
});

async function dissolveGroup(gid) {
  // 그룹의 모든 posts / messages / memberships / invites 삭제
  // FCM 알림 발송 (해체 알림)
  // 모든 토픽 구독자에게 unsubscribe 트리거
  // ... (별도 함수)
}

async function transferOwnership(gid, newOwnerId) {
  await updateDoc(doc(db, 'groups', gid), { ownerId: newOwnerId });
  await updateDoc(doc(db, 'memberships', `${newOwnerId}_${gid}`), { role: 'owner' });
  // 시스템 메시지 발송
  await addDoc(collection(db, 'groups', gid, 'messages'), {
    authorId: 'system',
    type: 'system',
    content: `소유자가 변경되었어요`,
    createdAt: serverTimestamp(),
  });
}
```

## Security Rule

```
match /users/{uid} {
  allow read: if request.auth.uid == uid
    // 그룹 멤버는 본인 그룹 내 다른 멤버 정보 조회 가능
    || exists(/databases/$(database)/documents/memberships/$(request.auth.uid + '_' + ...));
  allow update: if request.auth.uid == uid
    && request.resource.data.uid == uid
    && request.resource.data.phoneNumber == resource.data.phoneNumber  // 전화번호 변경 불가
    && request.resource.data.birthDate == resource.data.birthDate;     // 생년월일 변경 불가
  allow delete: if false; // Cloud Function만 가능
}

match /blocks/{compoundKey} {
  allow read: if request.auth.uid == resource.data.blockerId;
  allow create: if request.auth.uid == request.resource.data.blockerId
    && request.resource.data.blockerId != request.resource.data.blockedId;
  allow delete: if request.auth.uid == resource.data.blockerId;
}
```

## 인덱스
- `users`: `uid`, `phoneNumber` (자동)
- `memberships`: `uid`, `groupId`
- `groups`: `ownerId`
- `blocks`: `blockerId`, `blockedId`

## 분석 / 로깅
- 프로필 편집 빈도 (닉네임/사진)
- 차단 액션 / 해제 비율
- 계정 삭제율 (월별)
- 그룹 해체 vs 권한 위임 비율
- 평균 계정 삭제 소요 시간

## 보안
- 계정 삭제는 반드시 Cloud Function 경유 (트랜잭션 + 권한 검증)
- 프로필 사진 EXIF 제거
- 익명화된 메시지 작성자는 복구 불가능
