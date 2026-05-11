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

## hourSlot 산출 (D-12-01 확정)
- `src/lib/time.ts`에 `toHourSlot(date: Date): string` 유틸
- 출력: **UTC 기반** `YYYYMMDDHH` 문자열 (예: `2026051210`)
- 표시 시점에만 사용자 timezone (`users.timezone`)으로 변환
- 같은 그룹의 다른 timezone 멤버 공존 시에도 동일 슬롯 키 공유 (다중 timezone 안전)
- DST 처리:
  - Spring Forward: 로컬 02:00 슬롯이 표시 단계에서 사라짐 (UTC 키는 정상)
  - Fall Back: 로컬 02:00이 두 번 등장 → "(1차)/(2차)" 또는 UTC 병기로 구분

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

## 보안 룰 메모 (D-12-02 grace 30초 적용)
- `groups/{groupId}/posts` read: 요청자 uid가 `groups/{groupId}.memberIds`에 포함되어야 함
- `posts` write: 작성자 uid == 요청자 uid 그리고:
  - `request.time >= slotStart` (해당 슬롯 시작 시각 이후)
  - `request.time < slotStart + 1h + 30s` (다음 정각 + grace 30초 이내)
  - 소급 업로드 절대 금지
- 본인 영상 삭제: 작성 후 7일 내 본인만 가능 (Q-12-07)
- 본인 영상 교체(update): 슬롯 마감 전까지만 허용 (`request.time < slotStart + 1h + 30s`)

## 스트리밍 / CDN
- `videoUrl`은 Cloud Storage 직접 URL (gs://) 대신 다운로드 URL 사용
- 향후 사이즈 / 트랜스코딩 정책 결정 시 thumbnail URL과 짝지어 같이 저장

## 인덱스
- `groups/{groupId}/posts`: `hourSlot desc` (단일 필드 인덱스로 충분)
- (옵션) `authorId + hourSlot` 복합 인덱스 (멤버별 최신 검색에 사용)
