# capture — Data

## 입력
- `groupId: string`
- `slot: ISO timestamp` (옵션, 기본 = 현재 슬롯)
- 현재 사용자 uid

## 카메라 출력 (로컬 파일)
- 임시 URI: `file:///var/mobile/.../tmp/capture-{uuid}.mp4` (iOS) 또는 `/data/user/0/.../cache/capture-{uuid}.mp4` (Android)
- `expo-camera`의 `recordAsync` 옵션 (D-12-03, D-12-11, D-12-13):
  ```javascript
  await cameraRef.recordAsync({
    maxDuration: 3,
    codec: 'avc1',          // H.264 강제
    quality: '720p',        // 1280x720
    mute: false,
    mirror: false,          // 후면 카메라 기본
    // landscape 전용 — 디바이스 방향 체크는 진입 단계에서 처리
  });
  ```

## 디바이스 방향 잠금 (D-12-13)
```javascript
import * as ScreenOrientation from 'expo-screen-orientation';

// 화면 진입 시
await ScreenOrientation.lockAsync(ScreenOrientation.OrientationLock.LANDSCAPE);

// 화면 이탈 시
await ScreenOrientation.unlockAsync();
```

## 영상 메타 검증 (Cloud Function — D-12-13)
업로드 직후 Storage `finalize` 트리거에서 영상 메타 검증:
```javascript
exports.onVideoUploaded = onObjectFinalized(async (event) => {
  const filePath = event.data.name;
  if (!filePath.match(/^groups\/[^\/]+\/[^\/]+\/[^\/]+\.mp4$/)) return;
  
  // ffprobe 또는 ffmpeg로 메타 추출
  const metadata = await getVideoMetadata(filePath);
  
  // D-12-13: landscape 검증 (width > height)
  if (metadata.width <= metadata.height) {
    // portrait 영상 → 업로드 거부 (Storage 삭제 + Firestore 문서 삭제)
    await deleteObject(storageRef(storage, filePath));
    const [_, gid, slotKey, filename] = filePath.split('/');
    const uid = filename.replace('.mp4', '');
    await deleteDoc(doc(db, 'groups', gid, 'posts', `${slotKey}_${uid}`));
    
    // 사용자에게 알림
    await messaging().sendMulticast({
      tokens: userTokens,
      notification: {
        title: 'ChimeMe',
        body: '영상은 가로 방향으로 촬영해 주세요',
      },
      data: { type: 'video_rejected', reason: 'orientation' },
    });
    
    return;
  }
  
  // D-12-03: 해상도/코덱 검증 (옵션)
  // metadata.codec, metadata.width, metadata.height 확인
});
```

## 업로드 큐 (D-12-07)

### 큐 항목 스키마
```typescript
type UploadQueueItem = {
  id: string;           // uuid v4
  groupId: string;
  slotKey: string;      // UTC YYYYMMDDHH (D-12-01)
  uid: string;
  localUri: string;
  status: 'pending' | 'uploading' | 'success' | 'failed' | 'expired';
  retries: number;      // 0~5
  createdAt: number;    // Date.now()
  lastAttemptAt?: number;
  errorMessage?: string;
};
```

### AsyncStorage 키
- `upload_queue.v1`: `UploadQueueItem[]` (JSON)
- 모든 큐 변경은 트랜잭션처럼 처리 (read → modify → write)

### 백그라운드 업로드 흐름 (expo-task-manager)
```javascript
TaskManager.defineTask('UPLOAD_VIDEO_TASK', async () => {
  const queue = await getQueue();
  const pending = queue.filter(i => i.status === 'pending' || i.status === 'failed');
  
  for (const item of pending) {
    // 슬롯 마감 + grace 30초 초과 체크
    const slotEnd = parseSlotKey(item.slotKey) + 3600 * 1000 + 30 * 1000;
    if (Date.now() > slotEnd) {
      await updateQueueItem(item.id, { status: 'expired' });
      continue;
    }
    
    // 재시도 횟수 초과 체크
    if (item.retries >= 5) {
      await updateQueueItem(item.id, { status: 'failed' });
      continue;
    }
    
    // 업로드 시도
    try {
      await updateQueueItem(item.id, { status: 'uploading', lastAttemptAt: Date.now() });
      await uploadVideo(item);
      await updateQueueItem(item.id, { status: 'success' });
    } catch (e) {
      const backoff = Math.pow(2, item.retries) * 1000; // 1s, 2s, 4s, 8s, 16s
      setTimeout(() => {}, backoff);
      await updateQueueItem(item.id, {
        status: 'failed',
        retries: item.retries + 1,
        errorMessage: e.message,
      });
    }
  }
  
  return BackgroundFetch.BackgroundFetchResult.NewData;
});
```

### Storage 업로드 (D-12-07 멱등 경로)
```javascript
async function uploadVideo(item: UploadQueueItem) {
  const path = `groups/${item.groupId}/${item.slotKey}/${item.uid}.mp4`;
  const ref = storageRef(storage, path);
  const blob = await fetch(item.localUri).then(r => r.blob());
  
  await uploadBytesResumable(ref, blob, {
    contentType: 'video/mp4',
    cacheControl: 'public, max-age=604800', // 7일
  });
  
  const downloadUrl = await getDownloadURL(ref);
  
  // Firestore 문서 upsert (멱등)
  await setDoc(
    doc(db, 'groups', item.groupId, 'posts', `${item.slotKey}_${item.uid}`),
    {
      authorId: item.uid,
      groupId: item.groupId,
      hourSlot: item.slotKey,           // UTC YYYYMMDDHH
      hourSlotDate: parseSlotKey(item.slotKey), // Timestamp for queries
      videoPath: path,
      videoUrl: downloadUrl,
      createdAt: serverTimestamp(),
    }
  );
}
```

## 생성 데이터

### Firestore: `groups/{gid}/posts/{slotKey_uid}`
- 문서 ID는 `{slotKey}_{uid}` 결정적 키 (멱등성)
- `setDoc` upsert로 중복 방지 (D-12-07)
- Cloud Function 트리거: 새 post 생성 시 썸네일 생성 + FCM 알림 발송 (시점은 매시 정각 jitter)

### Storage: `groups/{gid}/{slotKey}/{uid}.mp4`
- 720p H.264 MP4
- 약 1 MB
- 7일 후 Storage Lifecycle Rule이 자동 삭제 (`age: 8`)

## Firestore Security Rule (capture 관련)
```
match /groups/{gid}/posts/{postId} {
  allow read: if request.auth.uid in get(/databases/$(database)/documents/groups/$(gid)).data.memberIds;
  
  allow create, update: if
    request.auth.uid == request.resource.data.authorId
    && request.auth.uid in get(/databases/$(database)/documents/groups/$(gid)).data.memberIds
    && request.resource.data.groupId == gid
    && postId == request.resource.data.hourSlot + '_' + request.auth.uid
    // D-12-02 grace 30초 적용 슬롯 시간 검증
    && (request.time.toMillis() < timestamp.value(request.resource.data.hourSlot).toMillis() + duration.value(1, 'h').toMillis() + duration.value(30, 's').toMillis());

  allow delete: if request.auth.uid == resource.data.authorId
    // 본인 영상은 슬롯 마감 후에도 7일 내 삭제 가능 (Q-12-07)
    && (resource.data.createdAt.toMillis() + duration.value(7, 'd').toMillis() > request.time.toMillis());
}
```

## Storage Security Rule
```
match /groups/{gid}/{slotKey}/{filename} {
  allow read: if request.auth.uid in firestore.get(/databases/(default)/documents/groups/$(gid)).data.memberIds;
  
  allow write: if request.auth.uid in firestore.get(/databases/(default)/documents/groups/$(gid)).data.memberIds
    && filename == request.auth.uid + '.mp4'
    && request.resource.size < 2 * 1024 * 1024  // 2MB 한도 (1MB 영상 + 여유)
    && request.resource.contentType == 'video/mp4';
}
```

## 인덱스
- `groups/{gid}/posts`: `hourSlotDate desc` (Timestamp 기반 정렬)
- (선택) `authorId + hourSlotDate` 복합 인덱스

## 클럭 동기화 (D-12-01 보완)
- 앱 시작 시 `serverTimestamp()` delta 계산
- delta > 30초이면 사용자 경고 + 슬롯 키 생성 시 보정

## 분석 / 로깅
- 업로드 성공률 (`uploaded / queued`)
- 평균 큐 → 업로드 완료 시간
- 만료된 큐 항목 비율 (`expired / total`)
- 재시도 횟수 분포 (히스토그램)
- 네트워크 끊김 → 재개 성공률

## 보안
- 로컬 임시 영상 파일은 업로드 성공 후 즉시 삭제
- 큐 영속화 데이터는 민감 정보 미포함 (URI만)
- Storage path 검증으로 타인 슬롯 덮어쓰기 차단
