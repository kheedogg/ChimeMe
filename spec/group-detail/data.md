# group-detail — Data

## 입력
- `groupId`
- (옵션) 시작 시간 슬롯 `slot: Date`

## 1차 쿼리 (그룹 정보 + 멤버)
```
groups/{groupId} → name, memberIds, ownerId, createdAt
users where uid in memberIds → displayName, photoUrl
```
- 결과는 컴포넌트 상태로 보관

## 2차 쿼리 (현재 슬롯 ~ ±N개 슬롯의 posts)
```
groups/{groupId}/posts
  where hourSlot in [현재 슬롯 - 3, ..., 현재 슬롯]   (페이지네이션 시 확장)
  orderBy hourSlot desc
```
- 멤버 × 슬롯 매트릭스로 재구성
  - `posts[slotISO][userId] = Post`

## hourSlot 산출
- `src/lib/time.ts`에 `toHourSlot(date: Date): Date` 유틸
- 입력 시각의 `minute`/`second`/`ms`를 0으로 만든 Date 반환
- 타임존: 사용자 로컬 (서버에도 동일 규칙 적용)

## 슬롯 개수 (페이지 인디케이터)
- 그룹 생성 시각부터 현재 시각까지의 시간 단위 차이
- **영상 보존 기간 7일(확정)** → 최대 노출 슬롯은 7 × 24 = 168개. 단, 인디케이터 점 168개를 모두 그리면 시인성이 떨어지므로 표시 방식은 별도 검토(예: 일 단위 그룹화, "오늘/어제/N일 전" 섹션화).

## 캐시
- 진입 시 fetch한 그룹/멤버는 화면이 살아있는 동안 유지
- 슬롯 이동 시 새 슬롯의 posts는 점진 fetch
- 시청한 슬롯은 in-memory에 보존

## unread 카운트 갱신
- 진입 시점에 `memberships/{uid}_{groupId}.unreadVideoCount = 0`
- 화면이 살아있는 동안 새 post가 들어오면(`onSnapshot`) 점진 추가하지만 unread는 증가시키지 않음

## 보안 룰 메모
- `groups/{groupId}/posts` read: 요청자 uid가 `groups/{groupId}.memberIds`에 포함되어야 함
- `posts` write: 작성자 uid == 요청자 uid + `hourSlot` ≤ 현재 슬롯 (소급 업로드 금지)

## 스트리밍 / CDN
- `videoUrl`은 Cloud Storage 직접 URL (gs://) 대신 다운로드 URL 사용
- 향후 사이즈 / 트랜스코딩 정책 결정 시 thumbnail URL과 짝지어 같이 저장

## 인덱스
- `groups/{groupId}/posts`: `hourSlot desc` (단일 필드 인덱스로 충분)
- (옵션) `authorId + hourSlot` 복합 인덱스 (멤버별 최신 검색에 사용)
