# group-settings — Data

## 입력
- `groupId: string`
- 현재 사용자 uid

## 사용 데이터

### Firestore 쿼리
```javascript
// 그룹 정보
const groupSnap = await getDoc(doc(db, 'groups', gid));

// 멤버 정보
const membershipsSnap = await getDocs(query(
  collection(db, 'memberships'),
  where('groupId', '==', gid)
));

// 각 멤버의 user 정보
const memberUids = membershipsSnap.docs.map(d => d.data().uid);
const usersSnap = await Promise.all(memberUids.map(uid => getDoc(doc(db, 'users', uid))));

// 활성 초대 링크
const invitesSnap = await getDocs(query(
  collection(db, 'invites'),
  where('groupId', '==', gid),
  where('expiresAt', '>', Timestamp.now()),
  orderBy('expiresAt', 'desc'),
  limit(1)
));
```

## 갱신 작업

### 그룹 이름/이모지 변경 (소유자만)
```javascript
await updateDoc(doc(db, 'groups', gid), {
  name: newName,
  emoji: newEmoji,
});

// 시스템 메시지
await addDoc(collection(db, 'groups', gid, 'messages'), {
  authorId: 'system',
  type: 'system',
  content: `그룹 이름이 '${newName}'으로 변경됐어요`,
  createdAt: serverTimestamp(),
});
```

### 초대 링크 재발급
```javascript
await runTransaction(db, async (tx) => {
  // 기존 활성 invites 만료 처리
  const activeSnap = await getDocs(query(
    collection(db, 'invites'),
    where('groupId', '==', gid),
    where('expiresAt', '>', Timestamp.now())
  ));
  activeSnap.docs.forEach(d => {
    tx.update(d.ref, { expiresAt: Timestamp.now() });
  });
  
  // 새 invite 생성
  const newInviteRef = doc(collection(db, 'invites'));
  tx.set(newInviteRef, {
    id: newInviteRef.id,
    groupId: gid,
    createdBy: user.uid,
    createdAt: serverTimestamp(),
    expiresAt: Timestamp.fromMillis(Date.now() + 7 * 24 * 60 * 60 * 1000),
    usedBy: [],
    maxUses: 7,
  });
});
```

### 멤버 추방 (Cloud Function)
```javascript
exports.kickMember = onCall(async (request) => {
  const { gid, targetUid, blockFromRejoining } = request.data;
  const callerUid = request.auth?.uid;
  
  // 권한 검증: caller는 소유자여야 함
  const groupSnap = await getDoc(doc(db, 'groups', gid));
  if (groupSnap.data().ownerId !== callerUid) {
    throw new HttpsError('permission-denied', 'Only owner can kick members');
  }
  
  await runTransaction(db, async (tx) => {
    // memberIds에서 제거
    tx.update(doc(db, 'groups', gid), {
      memberIds: arrayRemove(targetUid),
    });
    
    // memberships 삭제
    tx.delete(doc(db, 'memberships', `${targetUid}_${gid}`));
    
    // 재가입 차단 (옵션)
    if (blockFromRejoining) {
      tx.set(doc(db, 'blockedFromGroups', `${targetUid}_${gid}`), {
        uid: targetUid,
        groupId: gid,
        blockedBy: callerUid,
        createdAt: serverTimestamp(),
      });
    }
    
    // 시스템 메시지
    tx.set(doc(collection(db, 'groups', gid, 'messages')), {
      authorId: 'system',
      type: 'system',
      content: '한 멤버가 그룹에서 내보내졌어요',
      createdAt: serverTimestamp(),
    });
  });
  
  // FCM 토픽 구독 해제 (Admin SDK)
  const targetUserSnap = await getDoc(doc(db, 'users', targetUid));
  const tokens = targetUserSnap.data()?.fcmTokens || [];
  for (const token of tokens) {
    await messaging().unsubscribeFromTopic([token], `group_${gid}`);
  }
  
  // 추방자에게 FCM 알림
  await messaging().sendMulticast({
    tokens,
    notification: {
      title: 'ChimeMe',
      body: `'${groupSnap.data().name}'에서 내보내졌습니다`,
    },
    data: { type: 'kicked', gid },
  });
  
  return { success: true };
});
```

### 권한 위임 (Cloud Function)
```javascript
exports.transferOwnership = onCall(async (request) => {
  const { gid, newOwnerUid } = request.data;
  const callerUid = request.auth?.uid;
  
  const groupSnap = await getDoc(doc(db, 'groups', gid));
  if (groupSnap.data().ownerId !== callerUid) {
    throw new HttpsError('permission-denied', 'Only owner');
  }
  if (!groupSnap.data().memberIds.includes(newOwnerUid)) {
    throw new HttpsError('failed-precondition', 'New owner must be a member');
  }
  
  await runTransaction(db, async (tx) => {
    tx.update(doc(db, 'groups', gid), { ownerId: newOwnerUid });
    tx.update(doc(db, 'memberships', `${newOwnerUid}_${gid}`), { role: 'owner' });
    tx.update(doc(db, 'memberships', `${callerUid}_${gid}`), { role: 'member' });
    tx.set(doc(collection(db, 'groups', gid, 'messages')), {
      authorId: 'system',
      type: 'system',
      content: '소유자가 변경되었어요',
      createdAt: serverTimestamp(),
    });
  });
  
  return { success: true };
});
```

### 본인 탈퇴
```javascript
// 클라이언트
await runTransaction(db, async (tx) => {
  tx.update(doc(db, 'groups', gid), {
    memberIds: arrayRemove(user.uid),
  });
  tx.delete(doc(db, 'memberships', `${user.uid}_${gid}`));
  
  // 본인 채팅 메시지 authorId 익명화
  // 별도 Cloud Function이 onMembershipDeleted 트리거로 처리
  
  tx.set(doc(collection(db, 'groups', gid, 'messages')), {
    authorId: 'system',
    type: 'system',
    content: '한 멤버가 그룹을 떠났어요',
    createdAt: serverTimestamp(),
  });
});

await messaging().unsubscribeFromTopic(`group_${gid}`);
```

### 그룹 해체 (Cloud Function)
```javascript
exports.dissolveGroup = onCall(async (request) => {
  const { gid } = request.data;
  const callerUid = request.auth?.uid;
  
  const groupSnap = await getDoc(doc(db, 'groups', gid));
  if (groupSnap.data().ownerId !== callerUid) {
    throw new HttpsError('permission-denied', 'Only owner');
  }
  
  const memberIds = groupSnap.data().memberIds;
  
  // 1. 멤버 전원에게 FCM 알림
  for (const uid of memberIds) {
    const userSnap = await getDoc(doc(db, 'users', uid));
    const tokens = userSnap.data()?.fcmTokens || [];
    await messaging().sendMulticast({
      tokens,
      notification: {
        title: 'ChimeMe',
        body: `'${groupSnap.data().name}'이 해체되었습니다`,
      },
      data: { type: 'group_dissolved', gid },
    });
    for (const token of tokens) {
      await messaging().unsubscribeFromTopic([token], `group_${gid}`);
    }
  }
  
  // 2. 그룹 영상 일괄 삭제 (Storage + Firestore)
  const postsSnap = await getDocs(collection(db, 'groups', gid, 'posts'));
  for (const p of postsSnap.docs) {
    const path = p.data().videoPath;
    try { await deleteObject(storageRef(storage, path)); } catch (e) {}
    await deleteDoc(p.ref);
  }
  
  // 3. 채팅 메시지 일괄 삭제
  const messagesSnap = await getDocs(collection(db, 'groups', gid, 'messages'));
  const batch = writeBatch(db);
  messagesSnap.docs.forEach(d => batch.delete(d.ref));
  await batch.commit();
  
  // 4. memberships 일괄 삭제
  const membershipsSnap = await getDocs(query(
    collection(db, 'memberships'),
    where('groupId', '==', gid)
  ));
  const batch2 = writeBatch(db);
  membershipsSnap.docs.forEach(d => batch2.delete(d.ref));
  await batch2.commit();
  
  // 5. invites 일괄 삭제
  const invitesSnap = await getDocs(query(
    collection(db, 'invites'),
    where('groupId', '==', gid)
  ));
  const batch3 = writeBatch(db);
  invitesSnap.docs.forEach(d => batch3.delete(d.ref));
  await batch3.commit();
  
  // 6. 그룹 문서 삭제
  await deleteDoc(doc(db, 'groups', gid));
  
  // 7. blockedFromGroups는 그룹 해체 시 함께 정리
  const blockedSnap = await getDocs(query(
    collection(db, 'blockedFromGroups'),
    where('groupId', '==', gid)
  ));
  const batch4 = writeBatch(db);
  blockedSnap.docs.forEach(d => batch4.delete(d.ref));
  await batch4.commit();
  
  return { success: true };
});
```

### 본인 채팅 메시지 익명화 (Cloud Function onMembershipDeleted)
```javascript
exports.onMembershipDeleted = onDocumentDeleted('memberships/{key}', async (event) => {
  const { uid, groupId } = event.data.data();
  
  // 본인 채팅 메시지 authorId 익명화
  const myMessagesSnap = await getDocs(query(
    collection(db, 'groups', groupId, 'messages'),
    where('authorId', '==', uid)
  ));
  const anonymousId = `deleted_${hashUid(uid)}`;
  const batch = writeBatch(db);
  myMessagesSnap.docs.forEach(d => {
    batch.update(d.ref, { authorId: anonymousId });
  });
  await batch.commit();
});
```

## Security Rule

```
match /groups/{gid} {
  allow read: if request.auth.uid in resource.data.memberIds;
  
  allow update: if request.auth.uid in resource.data.memberIds
    && (
      // 일반 멤버: memberIds에서 본인 제거만 허용 (탈퇴)
      (request.resource.data.memberIds.size() == resource.data.memberIds.size() - 1
        && !(request.auth.uid in request.resource.data.memberIds))
      // 소유자: 모든 필드 수정 가능 (이름, 이모지, 멤버 추방)
      || resource.data.ownerId == request.auth.uid
    );
  
  allow delete: if false; // Cloud Function만
}

match /blockedFromGroups/{compoundKey} {
  allow read: if request.auth != null;
  allow write: if false; // Cloud Function만
}
```

## 인덱스
- `memberships`: `groupId` 단일 인덱스
- `invites`: `groupId + expiresAt desc`
- `blockedFromGroups`: 복합 키 자동

## 분석 / 로깅
- 그룹 이름/이모지 편집 빈도
- 초대 링크 재발급 빈도
- 멤버 추방 빈도 (사유 추적 필요 - 별도 reason 필드)
- 권한 위임 빈도
- 그룹 해체 vs 본인 탈퇴 비율
- 차단된 사용자 재초대 시도 빈도

## 보안
- 소유자 권한 검증은 Security Rule + Cloud Function 이중 검증
- 추방 / 해체 / 권한 위임은 모두 Cloud Function 경유 (트랜잭션 보장)
- 자기 자신 추방 차단 (`targetUid != callerUid`)
