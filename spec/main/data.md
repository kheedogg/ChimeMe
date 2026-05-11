# main — Data

## 사용 데이터

### Firestore 쿼리
```
groups
  where memberIds array-contains currentUser.uid
  orderBy lastActivityAt desc
```

### 카드 한 장에 필요한 필드
- `groups/{groupId}.name`
- `groups/{groupId}.memberIds.length` → 멤버 수
- `groups/{groupId}.lastActivityAt` → 상대 시간 변환
- `memberships/{uid}_{groupId}.unreadVideoCount` → 새 영상 뱃지 판정

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

## 새 영상 카운트 갱신
- 사용자가 카드 탭 → 진입 시 `memberships/{uid}_{groupId}.unreadVideoCount = 0`
- 새 post가 생기면 Cloud Function이 그룹 멤버의 memberships 카운트를 +1

## 인덱스 (사전 구성 필요)
- `groups` 컬렉션: `memberIds array_contains` + `lastActivityAt desc` 복합 인덱스

## 보안 룰 메모
- `groups` 문서 read: 요청자가 `memberIds`에 포함되어야 함
- `memberships/{compoundKey}` write: 본인 키만 가능
