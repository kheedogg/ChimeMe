# notifications — Data

## 입력
- 현재 사용자 uid

## 사용 데이터

### Firestore: `users/{uid}.notifications`
```typescript
type UserNotifications = {
  enabled: boolean;        // 전체 ON/OFF (기본 true)
  video: boolean;          // 영상 알림 (기본 true)
  chat: boolean;           // 채팅 알림 (기본 true)
  digestMode: boolean;     // 다이제스트 (기본 false, 3그룹+ 가입자만 사용 가능)
  quietHoursEnabled: boolean; // 방해금지 (기본 true)
  quietHoursStart: string;    // 'HH:MM' (기본 '23:00')
  quietHoursEnd: string;      // 'HH:MM' (기본 '07:00')
};
```

### Firestore: `users/{uid}.timezone`
- IANA timezone 문자열 (예: `Asia/Seoul`)
- onboarding에서 기기 timezone으로 초기화
- 사용자가 명시적으로 변경하지 않으면 자동 갱신 안 함 (해외여행 시 timezone 변경 의도치 않은 quiet hours 이동 방지)

### Firestore: `memberships/{uid}_{gid}.notificationsEnabled`
```typescript
type MembershipNotifications = {
  video: boolean; // 그룹별 영상 알림 (기본 true)
  chat: boolean;  // 그룹별 채팅 알림 (기본 true)
};
```

## 갱신 작업

### 사용자 설정 변경
```javascript
await updateDoc(doc(db, 'users', uid), {
  'notifications.video': false,
  updatedAt: serverTimestamp(),
});
```

### 그룹별 설정 변경
```javascript
await updateDoc(doc(db, 'memberships', `${uid}_${gid}`), {
  'notificationsEnabled.video': false,
});
```

## FCM 발송 시 체크 로직 (Cloud Function)

```javascript
exports.sendVideoNotification = onSchedule({
  schedule: '0 * * * *', // 매시 정각
  timeZone: 'UTC',
}, async (event) => {
  // 모든 활성 그룹 순회
  const groupsSnap = await getDocs(collection(db, 'groups'));
  
  for (const groupDoc of groupsSnap.docs) {
    const gid = groupDoc.id;
    
    // jitter (D-12-06: 0~120초 랜덤 지연)
    const jitter = Math.random() * 120 * 1000;
    setTimeout(() => sendGroupVideoNotification(gid), jitter);
  }
});

async function sendGroupVideoNotification(gid) {
  // 그룹 멤버 fetch
  const membershipsSnap = await getDocs(query(
    collection(db, 'memberships'),
    where('groupId', '==', gid)
  ));
  
  // 사용자별 필터
  const recipients = [];
  for (const m of membershipsSnap.docs) {
    const membership = m.data();
    if (!membership.notificationsEnabled?.video) continue;
    
    const userSnap = await getDoc(doc(db, 'users', membership.uid));
    const user = userSnap.data();
    
    // 전체 알림 OFF
    if (!user.notifications?.enabled) continue;
    // 영상 알림 OFF
    if (!user.notifications?.video) continue;
    
    // 방해 금지 시간 체크
    if (user.notifications?.quietHoursEnabled) {
      const inQuiet = isInQuietHours(
        user.notifications.quietHoursStart,
        user.notifications.quietHoursEnd,
        user.timezone
      );
      if (inQuiet) continue;
    }
    
    recipients.push(membership.uid);
  }
  
  // 다이제스트 모드 사용자는 별도 처리 (Cloud Scheduler가 통합 발송)
  const digestUsers = recipients.filter(/* user.notifications.digestMode === true */);
  const directUsers = recipients.filter(/* user.notifications.digestMode === false */);
  
  // 직접 발송: 토픽 메시지
  if (directUsers.length > 0) {
    await messaging().send({
      topic: `group_${gid}`,
      notification: {
        title: groupName,
        body: '새로운 시간이 시작됐어요!',
      },
      data: { gid, type: 'video', slotKey: getCurrentSlotKey() },
    });
  }
  
  // 다이제스트 사용자: 별도 카운터 누적 (다음 다이제스트 발송 시 합산)
  for (const uid of digestUsers) {
    await incrementDigestCounter(uid, gid);
  }
}

function isInQuietHours(start, end, timezone) {
  const now = new Date();
  const userTime = formatInTimeZone(now, timezone, 'HH:mm');
  
  if (start <= end) {
    // 같은 날 (예: 09:00 ~ 17:00)
    return userTime >= start && userTime < end;
  } else {
    // 자정 넘김 (예: 23:00 ~ 07:00)
    return userTime >= start || userTime < end;
  }
}
```

## 다이제스트 발송 (별도 스케줄)

```javascript
exports.sendDigestNotifications = onSchedule({
  schedule: '5 * * * *', // 매시 5분 (jitter 후 안정화 시점)
  timeZone: 'UTC',
}, async () => {
  // 다이제스트 카운터 fetch
  const countersSnap = await getDocs(query(
    collection(db, 'digestCounters'),
    where('count', '>', 0)
  ));
  
  for (const counter of countersSnap.docs) {
    const { uid, groupCount } = counter.data();
    const userSnap = await getDoc(doc(db, 'users', uid));
    const user = userSnap.data();
    
    // quiet hours 체크
    if (user.notifications?.quietHoursEnabled && 
        isInQuietHours(...)) continue;
    
    // 사용자의 FCM 토큰들에 다이제스트 발송
    await messaging().sendMulticast({
      tokens: user.fcmTokens,
      notification: {
        title: 'ChimeMe',
        body: `${groupCount}개 그룹에서 새 시간이 시작됐어요`,
      },
      data: { type: 'digest' },
    });
    
    // 카운터 리셋
    await deleteDoc(counter.ref);
  }
});
```

## 그룹 가입/탈퇴 시 토픽 구독 관리

### 가입 시 (D-12-06)
```javascript
await messaging().subscribeToTopic(`group_${gid}`);
```

### 탈퇴/강퇴 시 (D-12-06 + F-5 방어)
```javascript
// 클라이언트
await messaging().unsubscribeFromTopic(`group_${gid}`);
// 서버 측 보강: Cloud Function이 memberships 삭제 트리거에서 토픽 미구독 강제
exports.onMembershipDeleted = onDocumentDeleted('memberships/{key}', async (event) => {
  const { uid, groupId } = event.data.data();
  // FCM Admin SDK로 강제 토픽 구독 해제 (사용자 토큰 기반)
  const userSnap = await getDoc(doc(db, 'users', uid));
  const tokens = userSnap.data()?.fcmTokens || [];
  for (const token of tokens) {
    await messaging().unsubscribeFromTopic([token], `group_${groupId}`);
  }
});
```

## Security Rule

```
match /users/{uid} {
  allow update: if request.auth.uid == uid
    && (request.resource.data.diff(resource.data).affectedKeys()
        .hasOnly(['displayName', 'photoUrl', 'notifications', 'updatedAt', 'fcmTokens', 'timezone']));
}

match /memberships/{compoundKey} {
  allow update: if request.auth.uid == resource.data.uid;
}
```

## 인덱스
- `users`: 자동
- `memberships`: `uid + groupId` 자동 + `groupId` 단일 (Cloud Function 발송용)
- `digestCounters`: `count`, `uid`

## 분석 / 로깅
- 알림 종류별 ON/OFF 비율
- 방해 금지 사용률
- 다이제스트 사용률
- 그룹별 알림 OFF 사용 패턴
- FCM 발송 성공/실패율
- quiet hours로 인한 스킵 비율

## 보안
- 알림 설정 변경은 본인만 가능
- 다이제스트 카운터는 서버 쓰기 전용 (클라이언트 접근 금지)
