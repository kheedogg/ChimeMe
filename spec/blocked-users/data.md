# blocked-users — Data

## 입력
- 현재 사용자 uid

## 사용 데이터

### Firestore: `blocks/{blockerId_blockedId}`
```typescript
type Block = {
  blockerId: string;
  blockedId: string;
  createdAt: Timestamp;
  reason?: 'manual' | 'reported';   // 일반 차단 vs 신고 자동 차단 (Q-12-04)
};
```

### `users/{blockedId}` 정보
- `displayName`, `photoUrl`
- 탈퇴한 사용자의 경우 문서 부재 → "(탈퇴한 사용자)"로 표시

## 쿼리

### 차단 목록 fetch
```javascript
const blocksSnap = await getDocs(query(
  collection(db, 'blocks'),
  where('blockerId', '==', user.uid),
  orderBy('createdAt', 'desc')
));
```

### 차단 해제
```javascript
await deleteDoc(doc(db, 'blocks', `${myUid}_${theirUid}`));
```

## 실시간 구독
```javascript
const unsub = onSnapshot(
  query(
    collection(db, 'blocks'),
    where('blockerId', '==', user.uid),
    orderBy('createdAt', 'desc')
  ),
  (snap) => {
    setBlocks(snap.docs.map(d => ({ id: d.id, ...d.data() })));
  }
);
```

## 클라이언트 캐시 (다른 화면에서 활용)
- `zustand.blockStore`:
  ```typescript
  type BlockStore = {
    blockedUids: Set<string>;   // 차단한 사용자 uid 집합
    blockedByUids: Set<string>; // (선택) 본인을 차단한 사용자 uid 집합 — 클라이언트 불노출
  };
  ```
- 앱 시작 시 한 번 fetch + `onSnapshot`으로 실시간 갱신
- chat / group-detail 화면이 이 store를 참조하여 콘텐츠 필터링

## 차단 콘텐츠 필터링 패턴

### chat 메시지 필터링
```javascript
const visibleMessages = messages.filter(m => !blockStore.blockedUids.has(m.authorId));
```

### group-detail 영상 필터링
```javascript
const visibleClips = clips.filter(c => !blockStore.blockedUids.has(c.authorId));
```
- 차단 사용자의 영상 슬롯은 placeholder("미업로드")로 표시
- 또는 별도 마스킹 ("이 사용자를 차단했어요" — 향후 검토)

## Security Rule

```
match /blocks/{compoundKey} {
  allow read: if request.auth.uid == resource.data.blockerId;
  allow create: if request.auth.uid == request.resource.data.blockerId
    && request.resource.data.blockerId != request.resource.data.blockedId  // 자기 자신 차단 차단
    && exists(/databases/$(database)/documents/users/$(request.resource.data.blockedId));  // 존재하는 사용자만
  allow update: if false;
  allow delete: if request.auth.uid == resource.data.blockerId;
}
```

## 인덱스
- `blocks`: `blockerId + createdAt desc` 복합 인덱스
- `blocks`: 복합 키로 자동 인덱싱 (`{blockerId}_{blockedId}`)

## 계정 삭제 시 차단 정리 (D-12-09 연계)
계정 삭제 시 양방향 정리:
```javascript
// 본인이 차단한 목록 삭제
const myBlocksSnap = await getDocs(query(
  collection(db, 'blocks'),
  where('blockerId', '==', uid)
));
const batch1 = writeBatch(db);
myBlocksSnap.docs.forEach(d => batch1.delete(d.ref));
await batch1.commit();

// 본인을 차단한 목록도 삭제 (선택, 의미는 약하지만 정리 차원)
const blockedMeSnap = await getDocs(query(
  collection(db, 'blocks'),
  where('blockedId', '==', uid)
));
const batch2 = writeBatch(db);
blockedMeSnap.docs.forEach(d => batch2.delete(d.ref));
await batch2.commit();
```

## 분석 / 로깅
- 차단 액션 빈도 (수동 vs 신고 자동)
- 차단 해제율 (`unblocked / total_blocked`)
- 평균 차단 유지 기간
- 차단 사용자가 같은 그룹 멤버인 비율 (UX 개선 신호)

## 보안
- 차단 정보는 차단자에게만 노출 (피차단자는 본인이 차단됐는지 알 수 없음)
- 자기 자신 차단 차단 (Security Rule)
- 존재하지 않는 사용자 차단 차단 (Security Rule)
