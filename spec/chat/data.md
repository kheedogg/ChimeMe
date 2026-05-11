# chat — Data

## 입력
- `groupId: string`
- 현재 사용자 uid

## Firestore 컬렉션 구조 (D-12-08)
```
groups/{gid}/messages/{msgId}
  - clientMessageId: string  (멱등성 키, msgId와 동일)
  - authorId: string
  - content: string | null   (삭제 시 null)
  - videoQuote?: {
      postId: string,
      slotKey: string,
      authorId: string,
    }
  - createdAt: Timestamp (serverTimestamp)
  - deleted: boolean (default: false)
  - deletedAt?: Timestamp
  - hiddenByReports: boolean (default: false)  // Q-12-04 누적 3건 시 true
```

## 쿼리

### 초기 로드 (최신 50건)
```javascript
const q = query(
  collection(db, 'groups', gid, 'messages'),
  orderBy('createdAt', 'desc'),
  limit(50)
);
const snap = await getDocs(q);
const messages = snap.docs.map(d => ({ id: d.id, ...d.data() })).reverse();
```

### 실시간 구독 (신규 메시지)
```javascript
const unsub = onSnapshot(
  query(
    collection(db, 'groups', gid, 'messages'),
    orderBy('createdAt', 'desc'),
    limit(50)
  ),
  (snap) => { /* 메시지 상태 갱신 */ }
);
```

### 페이지네이션 (과거 메시지 로드)
```javascript
const q = query(
  collection(db, 'groups', gid, 'messages'),
  orderBy('createdAt', 'desc'),
  startAfter(oldestLoadedMessage.createdAt),
  limit(50)
);
```

## 메시지 발송 (D-12-08 멱등성)
```javascript
const clientMessageId = uuid();
await setDoc(
  doc(db, 'groups', gid, 'messages', clientMessageId),
  {
    clientMessageId,
    authorId: user.uid,
    content: text,
    videoQuote: videoQuoteData ?? null,
    createdAt: serverTimestamp(),
    deleted: false,
    hiddenByReports: false,
  }
);
```
- `setDoc`은 동일 `clientMessageId`로 재호출 시 덮어쓰기 → 재시도해도 중복 없음

## 메시지 삭제 (Q-12-05 — 5분 제한)
```javascript
await updateDoc(doc(db, 'groups', gid, 'messages', msgId), {
  content: null,
  deleted: true,
  deletedAt: serverTimestamp(),
});
```
- 문서는 보존하되 컨텐츠만 null로 만들고 deleted flag

## 메시지 신고 (Q-12-04)
```javascript
await addDoc(collection(db, 'reports'), {
  targetType: 'message',
  targetId: `${gid}/${msgId}`,
  reporterId: user.uid,
  reason: 'profanity' | 'inappropriate' | 'spam' | 'other',
  createdAt: serverTimestamp(),
  status: 'pending', // pending | reviewed | dismissed
});
```

Cloud Function 트리거 — 신고 누적 3건 시 메시지 자동 가림:
```javascript
exports.onReportCreated = onDocumentCreated('reports/{reportId}', async (event) => {
  const report = event.data.data();
  if (report.targetType !== 'message') return;
  
  // 동일 메시지의 신고 건수 카운트
  const countSnap = await getDocs(query(
    collection(db, 'reports'),
    where('targetId', '==', report.targetId),
    where('status', '==', 'pending')
  ));
  
  if (countSnap.size >= 3) {
    const [gid, msgId] = report.targetId.split('/');
    await updateDoc(doc(db, 'groups', gid, 'messages', msgId), {
      hiddenByReports: true,
    });
  }
});
```

## 사용자 차단 (Q-12-04)
```
blocks/{blockerId}_{blockedId}
  - blockerId: string
  - blockedId: string
  - createdAt: Timestamp
```
- 클라이언트가 자신의 `blocks` 컬렉션을 캐싱하여 메시지 필터링
- 차단된 사용자가 발송한 메시지는 클라이언트에서 숨김 (Firestore 쿼리는 차단 무관하게 전체 반환)

## 읽음 상태 (D-12-08)
```javascript
// 채팅 화면 진입 시
await updateDoc(doc(db, 'memberships', `${user.uid}_${gid}`), {
  lastReadMessageAt: serverTimestamp(),
});
```

"모두 읽음" 판정:
```javascript
// 그룹 멤버 전원의 memberships 읽기
const memberMembershipsSnap = await getDocs(query(
  collection(db, 'memberships'),
  where('groupId', '==', gid)
));
const allLastReadAt = memberMembershipsSnap.docs.map(d => d.data().lastReadMessageAt);
const latestMessageAt = messages[messages.length - 1].createdAt;
const allRead = allLastReadAt.every(t => t >= latestMessageAt);
```

## Firestore Security Rule

```
match /groups/{gid}/messages/{msgId} {
  function isMember() {
    return request.auth.uid in get(/databases/$(database)/documents/groups/$(gid)).data.memberIds;
  }
  
  allow read: if isMember();
  
  allow create: if isMember()
    && request.resource.data.authorId == request.auth.uid
    && msgId == request.resource.data.clientMessageId
    && request.resource.data.content.size() <= 2000;
  
  allow update: if request.auth.uid == resource.data.authorId
    && request.resource.data.deleted == true
    // Q-12-05: 5분 제한
    && (resource.data.createdAt.toMillis() + duration.value(5, 'm').toMillis() >= request.time.toMillis())
    && request.resource.data.content == null;
  
  allow delete: if false; // 직접 삭제 금지, soft delete만
}
```

## 채팅 보존 (Q-12-08 — 90일)
Cloud Scheduler (매일 03:00 UTC):
```javascript
exports.cleanupOldMessages = onSchedule('0 3 * * *', async () => {
  const cutoff = Timestamp.fromMillis(Date.now() - 90 * 24 * 60 * 60 * 1000);
  // 모든 그룹 순회
  const groupsSnap = await getDocs(collection(db, 'groups'));
  for (const groupDoc of groupsSnap.docs) {
    const oldMessagesSnap = await getDocs(query(
      collection(db, 'groups', groupDoc.id, 'messages'),
      where('createdAt', '<', cutoff),
      limit(500)
    ));
    const batch = writeBatch(db);
    oldMessagesSnap.docs.forEach(d => batch.delete(d.ref));
    await batch.commit();
  }
});
```

## 인덱스
- `groups/{gid}/messages`: `createdAt desc` (자동)
- `reports`: `targetId + status` 복합 인덱스
- `blocks`: 복합 키로 자동

## 캐시
- Firestore offline persistence 활성 (`enablePersistence`)
- 메시지 50건은 인메모리 캐시 (zustand `chatStore.messages[gid]`)
- 이미 로드된 메시지는 재요청하지 않음

## 분석 / 로깅
- 발송 메시지 수 (text vs video quote 비율)
- 평균 발송 → 수신 지연 (P50/P95)
- 신고 비율 + 사유별 분포
- 차단 액션 빈도
- 영상 인용 사용률
- 채팅 활성도 (그룹당 일일 평균 메시지 수)

## 보안
- 메시지 내용 길이 제한 (2000자)
- 클라이언트 임시 ID로 중복 방지
- 차단된 사용자 메시지는 클라이언트 필터 (서버 수준 차단은 P2 — 비용/복잡도 트레이드)
