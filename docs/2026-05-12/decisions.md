# 2026-05-12 의사결정 기록

> 3인 에이전트 팀 리뷰 결과를 바탕으로 5,000명 동시 사용자 기준 사이드 이팩트 방어를 위해 확정된 기술/정책 결정을 기록한다. 2026-05-11 결정과 충돌하는 항목은 본 문서가 우선한다.

---

## A. 확정 결정 사항 (3인 합의, 즉시 적용)

### D-12-01 슬롯 키 시간대 — UTC 저장 / 표시만 로컬 변환
- **결정**: 모든 슬롯 키(`hourSlot`)는 UTC 기반 `YYYYMMDDHH` 문자열로 Firestore에 저장. UI 표시 시점에만 사용자 로컬 timezone으로 변환.
- **근거**: 같은 그룹의 다른 timezone 멤버 공존 시 슬롯 분리 버그 방지. 3명 합의.
- **영향 파일**: `spec/group-detail/data.md:24-26` 수정 필요 ("사용자 로컬" 정의 폐기).
- **검증 방법**: `toHourSlot()` 단위 테스트 + DST Spring Forward/Fall Back 케이스.

### D-12-02 업로드 grace window — 정각 마감 +30초
- **결정**: 정각 도달 후 30초까지 직전 슬롯 업로드 허용. Security Rule에서 `request.time` 기준으로 판정.
- **근거**: 네트워크 지연 + 디바이스 클럭 오차 허용. UX 신뢰도와 데이터 정합성 균형.
- **검증 방법**: Emulator에서 정각 +0/+29/+30/+31초 4가지 업로드 시도.

### D-12-03 영상 인코딩 표준 — 720p H.264 30fps 2~3Mbps MP4
- **결정**: 모든 클라이언트에서 동일 인코딩으로 강제.
  - 코덱: H.264 (AVC)
  - 해상도: 1280×720
  - 프레임레이트: 30fps
  - 비트레이트: 2~3 Mbps
  - 컨테이너: MP4 (moov atom 선행)
- **근거**: HEVC는 Android 구버전 호환성 불균일. 1080p는 3초 영상에 과잉.
- **클립 추정 크기**: ~1 MB / 3초.

### D-12-04 App Check Day-1 적용
- **결정**: 베타 출시 시점부터 App Check 활성화.
  - iOS: DeviceCheck
  - Android: Play Integrity
  - Firestore / Storage / Functions / Auth 전 영역 토큰 검증 의무화
  - 디버그 빌드: debug provider 사용
- **근거**: SMS toll fraud / Firestore 무단 접근 방어.

### D-12-05 핫 도큐먼트 회피 — lastReadAt 기반 클라이언트 계산
- **결정**: `groups/{gid}.lastActivityAt` 및 `memberships/{uid}_{gid}.unreadVideoCount` 직접 쓰기 폐기.
  - `memberships`에 `lastReadVideoAt: Timestamp` 필드만 유지
  - 안읽음 카운트는 클라이언트가 `posts` 쿼리 결과를 `lastReadVideoAt` 기준으로 계산
  - 그룹 정렬 기준은 `posts/lastPost.createdAt` 또는 클라이언트 캐시로 대체
- **근거**: 5,000 DAU 정각 폭주 시 Firestore 1초 1문서 쓰기 제한 회피.
- **영향 파일**: `spec/main/data.md`, `spec/main/interactions.md` 수정 필요.

### D-12-06 FCM 발송 전략 — 토픽 메시징 + jitter + min instances
- **결정**:
  - 그룹별 토픽 구독 (`group_{gid}`)
  - 정각 발송은 Cloud Function이 0~120초 랜덤 jitter 적용
  - Cloud Function `minInstances: 2~3` 설정 (cold start 방지)
  - iOS 영상 알림은 `visible` (사일런트 푸시 빈도 제한 회피)
  - Android `notification.tag = group_{gid}`, iOS `threadIdentifier = group_{gid}` 그룹화
- **근거**: 1,800 그룹 정각 동시 발송 thundering herd 방지.

### D-12-07 업로드 큐 — expo-task-manager + AsyncStorage 영속화
- **결정**:
  - 큐 항목 상태: `pending` / `uploading` / `success` / `failed` / `expired`
  - 지수 백오프: 1s → 2s → 4s → 8s → 16s, 최대 5회
  - 슬롯 마감 + grace 30초 초과 시 큐에서 자동 제거 (`expired`)
  - Storage 경로: `{groupId}/{slotId}/{uid}.mp4` 고정 (멱등성 보장)
  - Firestore 문서: `setDoc(deterministicId)` (upsert)
- **근거**: 오프라인 → 온라인 복귀 시 자동 재시도 + 중복 업로드 방지.

### D-12-08 채팅 데이터 모델 — Firestore 서브컬렉션 + serverTimestamp
- **결정**:
  - 경로: `groups/{gid}/messages/{msgId}`
  - 정렬: `createdAt` (serverTimestamp 기반)
  - 페이지네이션: 50건/페이지 (최신 → 과거 방향)
  - 클라이언트 임시 ID (`clientMessageId`) 로 멱등성 보장 (재전송 중복 방지)
  - 읽음 표시: `memberships/{uid}_{gid}.lastReadMessageAt` 갱신, 클라이언트가 8개 문서 비교
- **근거**: 외부 채팅 SDK 도입 없이 5,000 DAU 규모 처리 가능. 읽음 fan-out 제거.

### D-12-09 계정 삭제 시 데이터 정리 파이프라인
- **결정**: 계정 삭제 트리거 → Cloud Function이 다음 순서로 정리.
  1. `users/{uid}` 문서 삭제
  2. `memberships/{uid}_*` 모든 문서 삭제 (소속 그룹 멤버에서 제외)
  3. Storage 영상 `gs://*/{groupId}/{slotId}/{uid}.mp4` 모두 삭제
  4. 채팅 메시지의 `authorId`를 `deleted_{uid_hash}` 토큰으로 익명화 (메시지 자체는 보존)
  5. 소유 그룹 처리: Q-12-06 결정에 따름 (현재 추천: 그룹 해체 강제 선택)
- **근거**: Apple 가이드라인 4.5.4 + GDPR 준수.

### D-12-10 Storage 자동 삭제 — 이중 파이프라인
- **결정**:
  - 1차: Cloud Storage Lifecycle Rule `age: 8` (7일 보존 + 1일 버퍼)
  - 2차: Cloud Scheduler (매일 03:00 UTC) → Cloud Function이 7일 이전 Firestore `posts/clips` 문서 배치 삭제 (500건 단위)
  - 두 파이프라인 중 하나가 실패해도 다른 쪽이 정리하도록 redundancy 확보
- **근거**: 단일 파이프라인 실패 시 비용 폭증 (월 $300+) 방지.

### D-12-11 영상 코덱 강제 변환 위치
- **결정**: `expo-camera`의 `recordAsync({ codec: 'avc1', maxDuration: 3 })` 옵션으로 캡처 시점부터 H.264 강제. 별도 변환 단계 없음.
- **근거**: 디바이스 기본 코덱(HEVC) 의존 시 Android 구버전 재생 불가 위험.

### D-12-12 SMS 비용 보호 — IP/번호 rate limit
- **결정**:
  - 동일 IP: 1시간 최대 5건 OTP 발송
  - 동일 전화번호: 1일 최대 10건 OTP 발송
  - reCAPTCHA Enterprise fallback 연동
  - 한도 초과 시 24시간 차단 + 관리자 알림
- **근거**: SMS toll fraud 비용 폭탄 방어 (App Check 우회 시도 대비).

---

## B. 사용자 결정 대기 항목 (Q-12-01 ~ Q-12-12)

> 본 항목들은 답변 수령 후 본 문서 §A로 이동한다.

### Q-12-01 1인당 최대 그룹 수
- **추천**: 10개
- **사유**: 무제한 시 1인당 Firestore 읽기 / FCM 토픽 구독 무한 증가. 비용/성능 균형점.

### Q-12-02 영상 보존 기간
- **현황**: docs/2026-05-11 decisions.md:51 = **7일**
- **추천**: 7일 유지
- **사유**: D-12-10 자동 삭제 파이프라인 설계 기준값.

### Q-12-03 미성년자 정책
- **추천**: 한국 우선 출시 시 KISA 가이드라인 준수 → **만 14세 이상만 가입 허용**, 생년월일 온보딩에서 수집
- **글로벌 출시 시**: 미국 13세 미만 COPPA 대응 추가 필요

### Q-12-04 UGC 신고/차단 정책
- **추천**:
  - 신고 즉시 **신고자 화면에서만** 가림 (P0)
  - 동일 콘텐츠 누적 3건 신고 시 **그룹 전체** 가림 + 관리자 검토 큐 등록
  - 허위 신고 5건 누적 시 신고 기능 24시간 정지
- **사유**: Apple 가이드라인 1.2 준수 + 어뷰징 방어 균형.

### Q-12-05 메시지 삭제 시간 제한
- **추천**: **5분** (Slack 기본값 참고)
- **사유**: 사용자 실수 정정 충분 + 악용 방지.

### Q-12-06 그룹 라이프사이클 정책
- **추천**:
  - **탈퇴자 영상**: 그룹에 잔존 (작성자 표시는 "(탈퇴한 사용자)"로 익명화)
  - **강퇴자 채팅 메시지**: placeholder 유지 ("(강퇴된 사용자)")
  - **소유자 탈퇴**: 자동 이전 없음 → **그룹 해체 강제 선택 UI** 노출 (탈퇴 vs 해체 vs 취소)

### Q-12-07 본인 영상 삭제/교체 정책
- **추천**:
  - 슬롯 마감 전: **무제한 교체** (같은 슬롯 덮어쓰기)
  - 슬롯 마감 후 ~ 7일 보존 중: **삭제만 가능, 교체 불가**
  - 7일 후: 자동 삭제

### Q-12-08 채팅 보존 기간
- **추천**: **90일**
- **사유**: 영구 보관 시 Firestore 문서 누적 비용 부담. 90일이면 대부분 유스케이스 커버.

### Q-12-09 푸시 알림 기본 설정
- **추천**:
  - 신규 가입 시 알림 **기본 ON**
  - 가입 그룹 3개 이상 시 **다이제스트 모드** 옵션 노출 (사용자가 선택)
  - 방해 금지 시간 기본값: 23:00 ~ 07:00 (KST 기준)

### Q-12-10 그룹 초대 방식
- **추천 (MVP)**: SMS 딥링크 + QR 코드
- **추천 (P2)**: 카카오톡 공유 (한국 사용자 우선)
- **초대 링크 유효기간**: 7일

### Q-12-11 지원 국가 / SMS 화이트리스트
- **추천 (베타)**: 한국 only (+82)
- **추천 (상용)**: 단계적 확대. 신규 국가 추가 시 SMS 비용 재추정 + reCAPTCHA Enterprise 확인

### Q-12-12 약관 동의 시점/방식
- **추천**:
  - `auth-phone` 화면 하단 체크박스 (전화번호 입력 후 OTP 발송 전)
  - 필수: 이용약관, 개인정보처리방침
  - 선택: 마케팅 정보 수신 동의
  - 약관 변경 시: 앱 시작 시 재동의 모달

---

## C. 2026-05-11 결정 사항과의 충돌 점검

| 2026-05-11 결정 | 2026-05-12 결정 | 결과 |
|---|---|---|
| `decisions.md:49` 그룹 최대 8명 | (변경 없음) | 유지 |
| `decisions.md:50` 매시 60분 윈도우 | D-12-02 grace +30초 추가 | 보강 |
| `decisions.md:51` 영상 7일 보존 | D-12-10 (Q-12-02 추천 = 7일 유지) | 호환 |
| `decisions.md:71` 푸시 알림 정책 미정 | D-12-06 + Q-12-09 (대기) | 부분 결정 |
| `decisions.md:72` 그룹 초대 방식 미정 | Q-12-10 (대기) | 결정 대기 |
| `decisions.md:73` 1인 최대 그룹 수 미정 | Q-12-01 (대기) | 결정 대기 |
| `decisions.md:74` `hourSlot` 타임존 미정 | D-12-01 UTC 저장 | **확정 (가장 중요)** |

---

## D. 결정 영향으로 수정 필요한 spec 파일

| 파일 | 수정 내용 | 관련 결정 |
|---|---|---|
| `spec/group-detail/data.md:24-26` | "사용자 로컬" 정의 폐기 → "UTC 저장, 표시만 로컬" | D-12-01 |
| `spec/group-detail/data.md:43` | Security Rule 의사코드에 grace 30초 명시 | D-12-02 |
| `spec/main/data.md:16` | `unreadVideoCount` 필드 제거, `lastReadVideoAt` 추가 | D-12-05 |
| `spec/main/interactions.md` | 안읽음 카운트 = 클라이언트 계산으로 변경 | D-12-05 |
| `spec/capture/overview.md:21-23` | 인코딩 표준 720p H.264 30fps 2~3Mbps 명시 | D-12-03 |
| `spec/capture/overview.md:27-30` | 큐 상태 enum + grace 30초 + 멱등 경로 명시 | D-12-07 |
| `spec/chat/overview.md:27-28` | 페이지네이션 50건, 삭제 5분 (Q-12-05 확정 시) | D-12-08, Q-12-05 |
| `spec/notifications/overview.md:14` | 방해금지 기본값 23:00~07:00 (Q-12-09 확정 시) | Q-12-09 |
- 추가 신규 spec 폴더: `report/`, `group-settings/`, `blocked-users/`, `account-delete/`, `legal/`

---

## E. 미해결 / 추후 검토

- SIM 스왑 방어 강화 (새 기기 로그인 시 기존 세션 만료) — 베타 후 상용 진입 전 결정
- 자동 모더레이션 도입 시점 (Cloud Vision API 등) — 베타 운영 데이터 기반 결정
- Cloud CDN 도입 시점 (Storage 비용 추이 모니터링 후) — 월 $200 초과 시점 도입 검토
- GDPR 데이터 내보내기 기능 — 유럽 출시 시 결정
- 분산 카운터 패턴 도입 — D-12-05로 우회했으나 채팅 확장 시 재검토
