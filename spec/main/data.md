# main — Data

## 사용 데이터

### Firestore 쿼리
```
groups
  where memberIds array-contains currentUser.uid
```
정렬은 클라이언트에서 수행 (D-12-05). 정렬 기준:
1. 새 영상 있는 그룹 우선
2. 최근 활동 시간 desc (각 그룹의 최신 post createdAt — 클라이언트 캐시 기반)

### 카드 한 장에 필요한 필드
- `groups/{groupId}.name`
- `groups/{groupId}.emoji`
- `groups/{groupId}.memberIds.length` → 멤버 수
- `memberships/{uid}_{groupId}.lastReadVideoAt` → 새 영상 뱃지 판정 (D-12-05)
- 최신 post 정보: 그룹 진입 시 또는 점진 fetch로 별도 쿼리 (`posts orderBy createdAt desc limit 1`)
  - 또는 클라이언트 캐시 (`zustand`)에서 마지막 본 슬롯 시각 보관

> **변경 이유 (D-12-05)**: `unreadVideoCount` 직접 쓰기는 5,000 DAU 정각 폭주 시 Firestore 1초 1문서 쓰기 제한과 충돌. 안읽음은 클라이언트가 `posts.createdAt > lastReadVideoAt` 비교로 계산.

## 캐시 / 실시간성
- 진입 시 한 번 fetch, 이후 그룹 변동 알림은 FCM 또는 Firestore listener로 갱신
- pull-to-refresh로 강제 재조회
- 메모리 캐시는 zustand 슬라이스(`store/groups.ts`)에 보관

## 상대시간 포맷
- "1분 미만" → "방금 전"
- "1~59분" → "N분 전"
- "1~23시간" → "N시간 전"
- "1일 이상" → "N일 전"
- `src/lib/time.ts`에 `formatRelative(date)` 유틸 작성

## 새 영상 판정 (D-12-05)
- 사용자가 그룹 카드 탭 → 진입 시 `memberships/{uid}_{groupId}.lastReadVideoAt = serverTimestamp()`
- 새 영상 존재 여부 판정 (클라이언트):
  ```typescript
  const hasNewVideo = group.lastPostAt > membership.lastReadVideoAt;
  ```
- `group.lastPostAt`은 `groups/{gid}/posts` 쿼리 결과의 최대 `createdAt`을 클라이언트가 캐싱
- Cloud Function이 fan-out 쓰기를 하지 않으므로 정각 폭주에 안전

## 인덱스 (사전 구성 필요)
- `groups` 컬렉션: `memberIds array_contains` + `lastActivityAt desc` 복합 인덱스

## 보안 룰 메모
- `groups` 문서 read: 요청자가 `memberIds`에 포함되어야 함
- `memberships/{compoundKey}` write: 본인 키만 가능
