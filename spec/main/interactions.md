# main — Interactions

## 진입
- 인증 상태 확인 → 미인증이면 `auth-phone`으로 리다이렉트
- 인증 완료 후 그룹 목록 fetch

## 탭 동작
| 영역 | 동작 |
|---|---|
| 그룹 카드 | `group-detail/[id]`로 이동, 진입 시 가장 최근 시간 슬롯 활성화 |
| 새 영상 뱃지가 있는 카드 | 동일하게 그룹 상세로 이동하지만 진입 시점에 unread 카운트를 0으로 마킹 |
| + FAB | `group-create` 모달 시트 오픈 |
| 프로필 아이콘 | `profile` 라우트로 이동 |

## 스크롤
- 카드가 많아지면 세로 스크롤. iOS bounce 활성.
- Pull-to-refresh로 그룹 목록 재조회.

## 정렬 (클라이언트 측 — D-12-05)
- 1차: 새 영상 있는 그룹 우선
- 2차: 각 그룹의 최신 post `createdAt` desc (없으면 `groups.createdAt`)
- (옵션) 사용자가 핀 고정 가능 — 추후

## 새 영상 판정 (D-12-05)
- 클라이언트 계산: 각 그룹의 최신 post `createdAt`이 `memberships.lastReadVideoAt`보다 크면 새 영상 표시
- 사용자 진입 시 `memberships.lastReadVideoAt = serverTimestamp()` 갱신
- Firestore 쓰기 fan-out 회피 (5,000 DAU 정각 폭주 안전)

## 에러 처리
- 네트워크 오류 → 인라인 배너 + "다시 시도" 버튼
- 권한 오류 → 강제 로그아웃 후 `auth-phone`

## 빈 상태
- 가입 그룹 0개 → 빈 상태 컴포넌트. "그룹 만들기" CTA가 + FAB과 동일 액션 트리거.

## 접근성
- 카드 단위 a11y label: "<그룹명>, 멤버 <N명>, <상대시간> 활동, <새 영상 있음/없음>"
- 새 영상 점은 단순 장식이 아니라 a11y 라벨에 포함

## 푸시 알림 연동 (향후)
- 매시 정각 알림에서 특정 그룹 deeplink 진입 시 메인을 건너뛰고 바로 `group-detail/[id]`로 이동
