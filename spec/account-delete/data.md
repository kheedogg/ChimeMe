# account-delete — Data

## 입력
- 현재 사용자 uid
- 사용자 선택: `OwnerGroupDecision[]`

## 데이터 모델

```typescript
type OwnerGroupDecision = {
  groupId: string;
  action: 'dissolve' | 'transfer';
  newOwnerId?: string;
};
```

## 사용 데이터 (사전 fetch)

### 소유 그룹 조회
```javascript
const ownerGroupsSnap = await getDocs(query(
  collection(db, 'groups'),
  where('ownerId', '==', user.uid)
));
```

### 각 그룹의 멤버 조회 (위임 후보)
```javascript
const membersSnap = await getDocs(query(
  collection(db, 'memberships'),
  where('groupId', '==', gid)
));
const otherMembers = membersSnap.docs
  .filter(m => m.data().uid !== user.uid)
  .map(m => m.data());
```

## Cloud Function: `deleteAccount` (단계별 idempotent)

```javascript
exports.deleteAccount = onCall(async (request) => {
  const uid = request.auth?.uid;
  if (!uid) throw new HttpsError('unauthenticated', 'Auth required');
  
  const { ownerGroupDecisions } = request.data;
  
  // === Stage 1: 소유 그룹 처리 ===
  const ownerGroupsSnap = await getDocs(query(
    collection(db, 'groups'),
    where('ownerId', '==', uid)
  ));
  
  // 결정 누락 검증
  const decidedGids = new Set(ownerGroupDecisions.map(d => d.groupId));
  for (const g of ownerGroupsSnap.docs) {
    if (!decidedGids.has(g.id)) {
      throw new HttpsError('invalid-argument', `Decision missing for group ${g.id}`);
    }
  }
  
  for (const decision of ownerGroupDecisions) {
    if (decision.action === 'dissolve') {
      await dissolveGroupInternal(decision.groupId);
    } else if (decision.action === 'transfer') {
      if (!decision.newOwnerId) {
        throw new HttpsError('invalid-argument', 'newOwnerId required for transfer');
      }
      await transferOwnershipInternal(decision.groupId, uid, decision.newOwnerId);
    }
  }
  
  // === Stage 2: 본인 영상 일괄 삭제 ===
  const myMembershipsSnap = await getDocs(query(
    collection(db, 'memberships'),
    where('uid', '==', uid)
  ));
  
  for (const m of myMembershipsSnap.docs) {
    const gid = m.data().groupId;
    
    const myPostsSnap = await getDocs(query(
      collection(db, 'groups', gid, 'posts'),
      where('authorId', '==', uid)
    ));
    
    for (const p of myPostsSnap.docs) {
      const path = p.data().videoPath;
      if (path) {
        try {
          await deleteObject(storageRef(storage, path));
        } catch (e) {
          // 이미 삭제된 경우 ignore
          console.warn(`Storage delete failed: ${path}`, e.message);
        }
      }
      await deleteDoc(p.ref);
    }
    
    // 그룹에서 본인 제거
    await updateDoc(doc(db, 'groups', gid), {
      memberIds: arrayRemove(uid),
    });
  }
  
  // === Stage 3: 채팅 메시지 익명화 ===
  const anonymousId = `deleted_${hashUid(uid)}`;
  
  for (const m of myMembershipsSnap.docs) {
    const gid = m.data().groupId;
    
    let lastDoc = null;
    while (true) {
      const q = lastDoc
        ? query(
            collection(db, 'groups', gid, 'messages'),
            where('authorId', '==', uid),
            startAfter(lastDoc),
            limit(500)
          )
        : query(
            collection(db, 'groups', gid, 'messages'),
            where('authorId', '==', uid),
            limit(500)
          );
      const snap = await getDocs(q);
      if (snap.empty) break;
      
      const batch = writeBatch(db);
      snap.docs.forEach(d => {
        batch.update(d.ref, { authorId: anonymousId });
      });
      await batch.commit();
      
      if (snap.docs.length < 500) break;
      lastDoc = snap.docs[snap.docs.length - 1];
    }
  }
  
  // === Stage 4: 사용자 데이터 + 인증 삭제 ===
  
  // memberships 일괄 삭제
  const batch = writeBatch(db);
  myMembershipsSnap.docs.forEach(d => batch.delete(d.ref));
  await batch.commit();
  
  // blocks 양방향 정리
  const myBlocksSnap = await getDocs(query(
    collection(db, 'blocks'),
    where('blockerId', '==', uid)
  ));
  const blockedMeSnap = await getDocs(query(
    collection(db, 'blocks'),
    where('blockedId', '==', uid)
  ));
  const batch2 = writeBatch(db);
  myBlocksSnap.docs.forEach(d => batch2.delete(d.ref));
  blockedMeSnap.docs.forEach(d => batch2.delete(d.ref));
  await batch2.commit();
  
  // 프로필 사진 Storage 삭제
  try {
    await deleteObject(storageRef(storage, `profiles/${uid}.jpg`));
  } catch (e) {
    console.warn(`Profile delete: ${e.message}`);
  }
  
  // FCM 토큰 정리 (서브스크라이브된 모든 토픽에서)
  const userSnap = await getDoc(doc(db, 'users', uid));
  const tokens = userSnap.data()?.fcmTokens || [];
  for (const m of myMembershipsSnap.docs) {
    const gid = m.data().groupId;
    for (const token of tokens) {
      try {
        await messaging().unsubscribeFromTopic([token], `group_${gid}`);
      } catch (e) {}
    }
  }
  
  // reports 정리 (본인 작성한 신고)
  const myReportsSnap = await getDocs(query(
    collection(db, 'reports'),
    where('reporterId', '==', uid)
  ));
  const batch3 = writeBatch(db);
  myReportsSnap.docs.forEach(d => batch3.delete(d.ref));
  await batch3.commit();
  
  // users 문서 삭제
  await deleteDoc(doc(db, 'users', uid));
  
  // Firebase Auth 계정 삭제
  await admin.auth().deleteUser(uid);
  
  return { success: true };
});

async function dissolveGroupInternal(gid) {
  // group-settings/data.md §dissolveGroup 참조
  // 동일 로직
}

async function transferOwnershipInternal(gid, fromUid, toUid) {
  await runTransaction(db, async (tx) => {
    tx.update(doc(db, 'groups', gid), { ownerId: toUid });
    tx.update(doc(db, 'memberships', `${toUid}_${gid}`), { role: 'owner' });
    // 본인 memberships는 다음 단계에서 삭제되므로 여기서는 role 변경 안 함
    tx.set(doc(collection(db, 'groups', gid, 'messages')), {
      authorId: 'system',
      type: 'system',
      content: '소유자가 변경되었어요',
      createdAt: serverTimestamp(),
    });
  });
}

function hashUid(uid) {
  return crypto.createHash('sha256').update(uid).digest('hex').slice(0, 12);
}
```

## 단계별 진행 상태 (Cloud Function 진행 알림)

### 옵션 A: 단일 호출 + 결과만 반환 (현재 권장)
- 클라이언트는 호출 후 응답까지 대기
- 진행 상태는 클라이언트 추정 (단계별 평균 시간 기반 시뮬레이션)

### 옵션 B: Cloud Function이 단계마다 Firestore에 진행 상태 write
- 클라이언트가 `accountDeleteProgress/{uid}` 문서를 `onSnapshot`으로 구독
- 단계 완료 시 Cloud Function이 해당 문서 갱신
- 클라이언트 UI 실시간 반영

옵션 B가 UX 측면에서 우수하나 옵션 A로 시작하고 필요 시 옵션 B로 전환.

## Security Rule

```
match /users/{uid} {
  allow delete: if false;  // Cloud Function만 가능
}

match /groups/{gid} {
  allow update: if request.auth.uid in resource.data.memberIds
    // 본인 탈퇴: memberIds에서 본인만 제거
    && request.resource.data.memberIds.size() == resource.data.memberIds.size() - 1
    && !(request.auth.uid in request.resource.data.memberIds);
}
```

## 인덱스 (Cloud Function 쿼리 효율)
- `groups`: `ownerId` 단일 인덱스
- `memberships`: `uid` 단일 인덱스
- `groups/{gid}/posts`: `authorId + createdAt` 복합 인덱스
- `groups/{gid}/messages`: `authorId + createdAt` 복합 인덱스
- `reports`: `reporterId` 단일 인덱스
- `blocks`: `blockerId`, `blockedId` 각각 단일 인덱스

## 분석 / 로깅
- 계정 삭제율 (월별)
- 평균 처리 시간 (단계별)
- 실패율 (단계별 분류)
- 소유 그룹 처리 비율 (해체 vs 위임)
- 재시도율
- 사용자 활성도 분포 (이용 일수별 삭제율)

## 보안
- Cloud Function 권한: 본인만 호출 가능 (`request.auth.uid === uid`)
- 모든 단계 idempotent 처리
- 단계 실패 시 부분 상태 그대로 보존 (재시도 가능)
- Firebase Auth 삭제는 최후 단계 (이전 단계 실패 시 인증 유지)
- 로그에 uid 마스킹 (`d3****12`)
