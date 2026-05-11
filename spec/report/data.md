# report — Data

## 입력
- `targetType: 'video' | 'message' | 'user'`
- `targetId: string` (대상 유형에 따라 형식 상이)
- `reason: string` (사유 enum)
- `description?: string` (부연 설명, 선택)

## Firestore 컬렉션

### `reports/{reportId}`
```typescript
type Report = {
  id: string;
  targetType: 'video' | 'message' | 'user';
  targetId: string;          // 영상: "{gid}/posts/{postId}", 메시지: "{gid}/messages/{msgId}", 사용자: "{uid}"
  reporterId: string;
  reason: 'profanity' | 'inappropriate' | 'spam' | 'violence' | 'minor_protection' | 'other';
  description: string | null;
  createdAt: Timestamp;
  status: 'pending' | 'reviewed_hide' | 'reviewed_dismiss' | 'auto_hidden';
  reviewedBy?: string;       // 관리자 uid
  reviewedAt?: Timestamp;
};
```

### `users/{uid}` 신고 관련 필드
```typescript
{
  // ...
  reportCount: number;              // 본인이 한 신고 총 건수
  reportCountIn24h: number;         // 최근 24시간 신고 건수
  reportBlockedUntil?: Timestamp;   // 신고 기능 제한 만료 시각
  reportedAgainstCount?: number;    // 본인이 받은 신고 건수 (운영 참고)
}
```

## 신고 생성 흐름

### 클라이언트
```javascript
const reportRef = doc(collection(db, 'reports'));
await setDoc(reportRef, {
  id: reportRef.id,
  targetType,
  targetId,
  reporterId: user.uid,
  reason,
  description: description || null,
  createdAt: serverTimestamp(),
  status: 'pending',
});

// 사용자 신고 시 자동 차단
if (targetType === 'user') {
  await setDoc(doc(db, 'blocks', `${user.uid}_${targetId}`), {
    blockerId: user.uid,
    blockedId: targetId,
    createdAt: serverTimestamp(),
    reason: 'reported',
  });
}
```

### 서버 (Cloud Function) - Q-12-04 자동 처리
```javascript
exports.onReportCreated = onDocumentCreated('reports/{reportId}', async (event) => {
  const report = event.data.data();
  
  // 1. 신고자 카운터 갱신 + 허위 신고 패널티 체크
  await checkReporterPenalty(report.reporterId);
  
  // 2. 미성년자 보호 위반은 즉시 처리
  if (report.reason === 'minor_protection') {
    await autoHideTarget(report);
    await notifyAdminSlack(report, 'URGENT: 미성년자 보호 위반 신고');
    return;
  }
  
  // 3. 동일 targetId 신고 건수 카운트
  const sameSnap = await getDocs(query(
    collection(db, 'reports'),
    where('targetId', '==', report.targetId),
    where('status', 'in', ['pending', 'auto_hidden'])
  ));
  
  // 4. 3건 누적 시 자동 가림 (Q-12-04)
  if (sameSnap.size >= 3) {
    await autoHideTarget(report);
  }
});

async function autoHideTarget(report) {
  if (report.targetType === 'video') {
    const [gid, _, postId] = report.targetId.split('/');
    await updateDoc(
      doc(db, 'groups', gid, 'posts', postId),
      { hiddenByReports: true }
    );
  } else if (report.targetType === 'message') {
    const [gid, _, msgId] = report.targetId.split('/');
    await updateDoc(
      doc(db, 'groups', gid, 'messages', msgId),
      { hiddenByReports: true }
    );
  } else if (report.targetType === 'user') {
    // 사용자 자체는 가리지 않고 운영자에게 관리자 검토 큐 전달
    await notifyAdminSlack(report, '사용자 신고 누적');
  }
  
  // 모든 관련 reports의 status 갱신
  const batch = writeBatch(db);
  const allSnap = await getDocs(query(
    collection(db, 'reports'),
    where('targetId', '==', report.targetId),
    where('status', '==', 'pending')
  ));
  allSnap.docs.forEach(d => batch.update(d.ref, { status: 'auto_hidden' }));
  await batch.commit();
}

async function checkReporterPenalty(reporterId) {
  const cutoff = Timestamp.fromMillis(Date.now() - 24 * 60 * 60 * 1000);
  const recentSnap = await getDocs(query(
    collection(db, 'reports'),
    where('reporterId', '==', reporterId),
    where('createdAt', '>', cutoff)
  ));
  
  const userRef = doc(db, 'users', reporterId);
  
  // Q-12-04: 24시간 내 5건 누적 시 24시간 정지
  if (recentSnap.size >= 5) {
    // 5건 중 dismiss된 건수가 다수면 허위 신고로 간주
    const dismissedCount = recentSnap.docs.filter(d => d.data().status === 'reviewed_dismiss').length;
    if (dismissedCount >= 3) {
      await updateDoc(userRef, {
        reportBlockedUntil: Timestamp.fromMillis(Date.now() + 24 * 60 * 60 * 1000),
      });
    }
  }
  
  await updateDoc(userRef, {
    reportCount: increment(1),
    reportCountIn24h: increment(1),
  });
}
```

## 클라이언트 가림 처리

진입 시 `reports` 컬렉션에서 본인 신고 fetch + 메모리 캐싱:
```javascript
const myReportsSnap = await getDocs(query(
  collection(db, 'reports'),
  where('reporterId', '==', user.uid)
));
const reportedTargetIds = new Set(myReportsSnap.docs.map(d => d.data().targetId));

// 영상 / 메시지 표시 시 reportedTargetIds 체크
function isHidden(item) {
  return reportedTargetIds.has(item.targetId) || item.hiddenByReports === true;
}
```

## Security Rule

```
match /reports/{reportId} {
  allow read: if request.auth.uid == resource.data.reporterId;
  
  allow create: if request.auth.uid == request.resource.data.reporterId
    && request.resource.data.targetType in ['video', 'message', 'user']
    && request.resource.data.reason in ['profanity', 'inappropriate', 'spam', 'violence', 'minor_protection', 'other']
    && (request.resource.data.description == null || request.resource.data.description.size() <= 500)
    // 본인 콘텐츠 신고 차단
    && request.resource.data.targetId.split('/')[2] != request.auth.uid;
  
  allow update, delete: if false;  // 관리자 SDK만 가능
}
```

## 인덱스
- `reports`: `targetId + status` 복합 (자동 가림 카운트용)
- `reports`: `reporterId + createdAt desc` 복합 (개인 신고 이력 + 패널티 체크용)
- `reports`: `status + createdAt asc` (관리자 검토 큐 정렬)

## 분석 / 로깅
- 신고 카테고리별 분포
- 자동 가림율 (`auto_hidden / total`)
- 신고자 본인 차단율 (`blocked_users / total reporters`)
- 미성년자 보호 위반 신고 빈도 → 관리자 알림 트리거
- 신고 후 가림 결정까지 평균 시간

## 보안
- 신고는 익명 처리 (피신고자는 신고자 식별 불가)
- `reports` 컬렉션 read는 본인 신고만 가능
- 관리자 SDK 경유 검토만 status 갱신 가능
- 자동 가림은 동일 콘텐츠 3건 누적 시 즉시 적용
- 24시간 후 `reportCountIn24h` 자동 리셋 (Cloud Scheduler)
