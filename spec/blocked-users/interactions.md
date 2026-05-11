# blocked-users — Interactions

## 진입
- profile 화면 메뉴 "차단한 사용자 (N명)" 탭
- 진입 시 본인의 `blocks` 컬렉션 fetch

## 데이터 로드
```javascript
const blocksSnap = await getDocs(query(
  collection(db, 'blocks'),
  where('blockerId', '==', user.uid),
  orderBy('createdAt', 'desc')
));

const blockedUids = blocksSnap.docs.map(d => d.data().blockedId);
const usersSnap = await Promise.all(
  blockedUids.map(uid => getDoc(doc(db, 'users', uid)))
);
```

## 차단 해제 흐름
1. "해제" 탭 → 확인 다이얼로그 표시
2. "차단 해제" 탭:
   - 항목에 스피너 표시
   - Firestore: `deleteDoc(doc(db, 'blocks', `${myUid}_${theirUid}`))`
   - 메모리 캐시(`zustand.blockStore`) 갱신 → 해당 사용자 콘텐츠 가림 해제
3. 항목 fade-out 애니메이션 → 리스트에서 제거
4. 빈 상태 도달 시 빈 상태 컴포넌트로 전환

## 차단 사용자가 탈퇴한 경우
- `users/{theirUid}` 문서가 없으면 (계정 삭제됨):
  - 닉네임 자리에 "(탈퇴한 사용자)" 표시
  - 아바타 기본 회색
  - 해제 버튼은 그대로 표시 (해제 가능)

## 차단 해제 후 영향
- 클라이언트 메모리 캐시 즉시 갱신
- `chat` / `group-detail` 진입 시 가림 해제된 콘텐츠 다시 표시
- 차단 해제는 양방향이 아니므로 피차단자에게 알림 없음 (차단 사실 자체를 피차단자가 모름)

## 빈 상태 진입로
- 빈 상태에서 별도 액션 없음 (정보 화면)
- 향후: "사용자 차단 방법 안내" 도움말 링크 추가 가능

## 실시간 갱신
- `onSnapshot` 구독:
  - 다른 화면에서 새로운 차단 추가 시 자동 반영
  - 차단 해제 시 자동 반영

## 차단 사용자 일괄 해제 (P2)
- 베타에서는 미지원
- 향후: 화면 우상단 "모두 해제" 옵션 검토

## 에러 처리
- 해제 실패 → 토스트 + 항목 상태 복귀
- 네트워크 오류 → 인라인 배너 + 재시도 버튼

## 접근성
- 해제 진행/완료 a11y live region
- 빈 상태 a11y: "차단한 사용자 목록이 비어 있습니다"

## 트래킹
- `blocked_users_view`
- `user_unblocked` (theirUid)
- `blocked_users_empty_view` (빈 상태 진입)
