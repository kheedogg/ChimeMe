# group-create — Data

## 입력
- 현재 사용자 uid
- 그룹 이름 (string, 1~30)
- 그룹 이모지 (string, single grapheme cluster)

## 생성 데이터 (Firestore 트랜잭션)

```javascript
const newGroupRef = doc(collection(db, 'groups'));
const newMembershipRef = doc(db, 'memberships', `${user.uid}_${newGroupRef.id}`);
const newInviteRef = doc(collection(db, 'invites'));

await runTransaction(db, async (tx) => {
  tx.set(newGroupRef, {
    id: newGroupRef.id,
    name: groupName,
    emoji: groupEmoji,
    ownerId: user.uid,
    memberIds: [user.uid],
    maxMembers: 8,
    createdAt: serverTimestamp(),
    // D-12-05: lastActivityAt 직접 쓰기 회피 → 클라이언트 계산 또는 Cloud Function 비동기
  });

  tx.set(newMembershipRef, {
    uid: user.uid,
    groupId: newGroupRef.id,
    role: 'owner',
    joinedAt: serverTimestamp(),
    lastReadVideoAt: serverTimestamp(), // D-12-05: unreadVideoCount 대체
    lastReadMessageAt: serverTimestamp(), // D-12-08
    notificationsEnabled: true, // Q-12-09
  });

  tx.set(newInviteRef, {
    id: newInviteRef.id,
    groupId: newGroupRef.id,
    createdBy: user.uid,
    createdAt: serverTimestamp(),
    expiresAt: Timestamp.fromMillis(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7일
    usedBy: [],
    maxUses: 7, // 8명 - 본인 1명 = 7명 추가 가능
  });
});
```

## FCM 토픽 구독
```javascript
await messaging().subscribeToTopic(`group_${newGroupId}`);
```

## 초대 링크 형식
- URL: `https://chimeme.app/invite/{inviteId}`
- 딥링크 / Universal Link / App Link 모두 동일 URL 사용
- iOS: `apple-app-site-association` 파일에 `/invite/*` 등록
- Android: `assetlinks.json` + `intent-filter` 등록
- 미설치 사용자: 자동으로 스토어 리다이렉트 (Firebase Dynamic Links 대안 검토 — 2025년 8월 종료 예정이라 자체 구현 또는 Branch.io)

## 초대 링크 사용 시 (외부 진입 처리)
```javascript
// 가입 처리 Cloud Function
exports.redeemInvite = onCall(async (request) => {
  const { inviteId } = request.data;
  const uid = request.auth?.uid;
  if (!uid) throw new HttpsError('unauthenticated', 'Auth required');

  const inviteRef = doc(db, 'invites', inviteId);
  const groupRef = ...; // 트랜잭션 내에서 동적 참조

  return await runTransaction(db, async (tx) => {
    const inviteSnap = await tx.get(inviteRef);
    if (!inviteSnap.exists()) throw new HttpsError('not-found', 'Invite not found');
    const invite = inviteSnap.data();
    
    if (invite.expiresAt.toMillis() < Date.now()) {
      throw new HttpsError('failed-precondition', 'EXPIRED');
    }
    
    const groupRef = doc(db, 'groups', invite.groupId);
    const groupSnap = await tx.get(groupRef);
    const group = groupSnap.data();
    
    if (group.memberIds.length >= group.maxMembers) {
      throw new HttpsError('failed-precondition', 'GROUP_FULL');
    }
    if (group.memberIds.includes(uid)) {
      throw new HttpsError('already-exists', 'ALREADY_MEMBER');
    }
    
    // 차단 사용자 체크 (Q-12-06)
    const blockSnap = await tx.get(doc(db, 'blockedFromGroups', `${uid}_${invite.groupId}`));
    if (blockSnap.exists()) {
      throw new HttpsError('permission-denied', 'BLOCKED');
    }

    tx.update(groupRef, { memberIds: arrayUnion(uid) });
    tx.update(inviteRef, { usedBy: arrayUnion(uid) });
    tx.set(doc(db, 'memberships', `${uid}_${invite.groupId}`), {
      uid, groupId: invite.groupId, role: 'member',
      joinedAt: serverTimestamp(),
      lastReadVideoAt: serverTimestamp(),
      lastReadMessageAt: serverTimestamp(),
      notificationsEnabled: true,
    });
    
    return { groupId: invite.groupId };
  });
});
```

## 보안 룰

```
match /groups/{gid} {
  allow read: if request.auth.uid in resource.data.memberIds;
  allow create: if request.auth.uid == request.resource.data.ownerId
    && request.resource.data.memberIds == [request.auth.uid]
    && request.resource.data.maxMembers == 8
    && request.resource.data.name.size() >= 1
    && request.resource.data.name.size() <= 30;
  allow update: if request.auth.uid in resource.data.memberIds; // 가입/탈퇴/이름 변경 등
  allow delete: if request.auth.uid == resource.data.ownerId;
}

match /memberships/{compoundKey} {
  allow read: if request.auth.uid == resource.data.uid
    || (request.auth.uid in get(/databases/$(database)/documents/groups/$(resource.data.groupId)).data.memberIds);
  allow create: if request.auth.uid == request.resource.data.uid;
  allow update: if request.auth.uid == resource.data.uid;
  allow delete: if request.auth.uid == resource.data.uid
    || request.auth.uid == get(/databases/$(database)/documents/groups/$(resource.data.groupId)).data.ownerId;
}

match /invites/{inviteId} {
  allow read: if request.auth != null; // 누구나 초대 정보 조회 가능
  allow create: if request.auth.uid == request.resource.data.createdBy
    && request.auth.uid in get(/databases/$(database)/documents/groups/$(request.resource.data.groupId)).data.memberIds;
}
```

## 인덱스
- `groups`: `memberIds array_contains` (자동)
- `memberships`: 복합 키로 자동 인덱싱
- `invites`: `groupId + expiresAt` 복합 인덱스 (만료 정리용)

## 자동 정리 (Cloud Scheduler)
- 매일 03:00 UTC: 만료된 invites 문서 삭제 (500개 단위 배치)
- 미사용 invites는 7일 후 자동 삭제

## 부적절 단어 검증 (Cloud Function)
- 그룹 생성 직후 트리거
- `name` + `emoji`에 부적절 내용 포함 시 자동 마스킹 또는 그룹 자동 삭제 + 작성자 알림

## 분석 / 로깅
- 그룹 생성률
- 초대 링크 발급 → 사용 전환율
- SMS / QR / 복사 별 공유 액션 비율
- 만료된 링크 클릭 빈도 (7일 적절성 검증)
