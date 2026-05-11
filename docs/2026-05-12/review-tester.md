# ChimeMe QA 검토서 — Tester

> **작성자**: Test Engineer
> **작성일**: 2026-05-12
> **대상 규모**: 5,000명 동시 사용자 (DAU)
> **스택**: React Native + Expo + Firebase (Auth / Firestore / Storage / FCM)

---

## 핵심 요약 (5줄)

1. 매시 정각 업로드 마감 경계는 ChimeMe의 핵심 비즈니스 로직이며, 클럭 스큐·grace window·슬롯 키 timezone 3가지가 동시에 맞물리는 **가장 높은 밀도의 엣지케이스 클러스터**다.
2. 정각 직후 5,000 DAU × 6회 업로드 × fan-out 쓰기가 동시에 터지는 **Firestore 쓰기 폭주** 시나리오는 개발 전 데이터 모델 확정 없이는 통합 테스트 자체가 불가능하다.
3. review-developer.md에서 다루지 않은 **SIM 스왑, 추방 직후 채팅 메시지 전송, DST Fall-Back 중복 슬롯, 미성년자 온보딩 우회, GDPR 내보내기 누락** 5개 항목이 P0~P1 수준 위험을 내포한다.
4. UGC 신고/차단이 기획서에 전무한 상태는 **Apple 가이드라인 1.2 리젝** 직결 사안으로, 테스트 설계 이전에 기획 확정이 선행되어야 한다.
5. Firebase Emulator Suite + k6 부하 테스트 조합으로 정각 thundering herd를 로컬에서 재현 가능하며, 실기기 없이도 80% 이상의 엣지케이스를 사전 검증할 수 있다.

---

## §1 엣지케이스 카탈로그

### A. 인증 (전화번호/SMS)

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| A-1 | 잘못된 형식 전화번호 입력 (예: 01012345, 국가코드 없이) | `auth-phone`에서 유효성 검사 없이 "코드 받기" 탭 | Firebase signInWithPhoneNumber 오류 메시지가 한국어로 번역되지 않아 원시 오류 코드가 노출될 수 있음 | 높음 | 중 | Firebase Emulator에서 malformed 번호 5가지 유형을 단위 테스트. UI에서 인라인 에러 검증 |
| A-2 | 국제번호 입력 (+1, +81, +44) | 해외 거주 사용자, 해외여행 중 기기 설정 변경 | 기본 국가코드 +82 고정 시 국제번호 발송 실패. SMS 비용 국가별 상이 ($0.01~$0.06/건) | 중간 | 중 | 국가코드 선택 UI 구현 후 E2E 테스트. Firebase Emulator에서 각국 번호 형식 검증 |
| A-3 | 동일 전화번호 중복 가입 시도 | 기기 교체, 앱 재설치, 여러 기기에서 동시 로그인 | Firebase는 동일 번호로 기존 계정에 재로그인 처리. 기존 `users/{uid}` 문서가 있을 때 onboarding으로 분기하면 안 됨 (기존 사용자를 신규로 오인) | 높음 | 상 | Emulator에서 동일 번호로 두 번 인증. `users/{uid}` 존재 여부 체크 로직 단위 테스트 |
| A-4 | OTP 발송 폭주 (재전송 버튼 연타) | 사용자 실수 또는 악의적 자동화 | App Check 미적용 시 SMS 비용 폭탄. 쿨다운 미구현 시 60초 내 수십 건 발송 가능 | 낮음(악의) ~ 높음(실수) | 상 | 재전송 버튼 연타 10회 시도. 쿨다운 UI 잠금 및 백엔드 rate limit 동작 확인 |
| A-5 | SIM 스왑 공격 | 공격자가 통신사를 속여 피해자 번호 이전 후 OTP 수신 | Firebase Phone Auth는 SIM 스왑을 감지하지 않음. 공격자가 기존 계정 전체 접근 가능. 피해자 세션 강제 만료 없음 | 낮음(베타) ~ 중간(상용) | 상 | 보안 정책 문서화 테스트. 새 기기 + 동일 번호 로그인 시 기존 세션 만료 여부 확인 (현재 Firebase 기본동작: 만료 안 함 — 추가 구현 필요) |
| A-6 | App Check 우회 (루팅/탈옥 기기, 에뮬레이터에서 프로덕션 API 호출) | 루팅된 Android 또는 탈옥 iOS에서 앱 실행 | Play Integrity / DeviceCheck 실패 → App Check 토큰 발급 불가 → Firebase 요청 전체 거부. 정상 사용자도 루팅 기기면 차단됨 (사용자 경험 저하) | 중간 | 중 | 루팅 에뮬레이터에서 앱 실행. App Check enforcement 모드 vs debug 모드 전환 테스트 |
| A-7 | OTP 코드 만료 후 입력 | SMS 수신 지연(5분+) 후 코드 입력 | Firebase 기본 OTP 유효 시간 5분. 만료 후 입력 시 `auth/code-expired` 오류. 사용자에게 재전송 안내 없으면 앱이 멈춘 것처럼 보임 | 중간 | 중 | Emulator에서 만료된 OTP 입력 시나리오. `auth/code-expired` 오류 처리 UI 검증 |
| A-8 | 베타 화이트리스트 외 번호 진입 | 비초대 사용자가 앱 다운로드 후 인증 시도 | 화이트리스트 정책 미정 상태. 서버 측 차단 없으면 SMS 발송 후 사후 차단 → SMS 비용 낭비 | 중간 | 중 | 화이트리스트 확정 후, 미등록 번호로 전체 인증 흐름 실행. SMS 발송 전 차단 vs 발송 후 차단 비교 |

### B. 그룹 생성/가입/탈퇴/해체

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| B-1 | 2명이 동시에 동일 그룹 가입 (8명 그룹에 7명째, 8명째 동시 탭) | 초대 링크 공유 후 두 사람이 거의 동시에 "가입" 탭 | Firestore transaction 없이 `memberIds.length` 체크 시 9명이 가입될 수 있음 (TOCTOU race) | 중간 | 상 | Firebase Emulator에서 동시 가입 트랜잭션 테스트. `runTransaction`으로 원자적 증분 구현 여부 확인 |
| B-2 | 그룹 멤버 1명이 탈퇴하는 동시에 다른 멤버가 영상 업로드 | 탈퇴 처리와 업로드 Cloud Function 동시 실행 | 탈퇴 멤버의 `memberships` 문서 삭제 중 unreadVideoCount +1 쓰기가 충돌. 삭제된 멤버 문서에 고아 데이터 생성 가능 | 낮음 | 중 | Emulator에서 탈퇴 + 업로드 동시 실행 스크립트. 탈퇴 후 memberships 문서 완전 삭제 확인 |
| B-3 | 그룹 소유자(ownerId)가 탈퇴 시도 | 프로필 → 그룹 목록 → 탈퇴 | 소유권 이전 정책 미정. 자동 이전 로직 없으면 그룹이 ownerId 없는 상태로 orphan화. 다른 멤버가 그룹 관리 불가 | 높음(결국 발생) | 상 | 소유자 탈퇴 시도 플로우 전체 E2E 테스트. 소유권 이전 UI/로직 구현 전까지는 차단 처리가 맞는지 확인 |
| B-4 | 그룹 최대 인원(8명) 초과 초대 링크 진입 | 8명 꽉 찬 그룹의 초대 링크를 9번째 사람이 클릭 | 앱 미설치 사용자는 스토어 → 설치 → 딥링크 복원 → 가입 시도 → 8명 초과 오류. 설치까지 유도한 후 거부되는 나쁜 UX | 중간 | 중 | 8명 꽉 찬 그룹의 초대 링크로 신규 사용자 가입 E2E. 오류 메시지 및 대안 안내 UI 확인 |
| B-5 | 차단된 사용자를 초대 링크로 다시 초대 | 강퇴 후 차단된 멤버가 새 초대 링크 입수 | 차단/재초대 차단 정책 미정. 차단 로직 없으면 강퇴 직후 재가입 가능 | 낮음 | 상 | 강퇴 + 재가입 시도 시나리오. 차단 목록 조회 및 가입 거부 로직 단위 테스트 |
| B-6 | 그룹 최대 인원 경계에서 레이스 컨디션 | 7번째 멤버 가입 처리 중 8번째도 동시 진행 | B-1과 동일 메커니즘. Firestore `arrayUnion`은 원자적이나 배열 길이 체크는 별도 transaction 필요 | 중간 | 상 | Firestore 보안 룰에서 `resource.data.memberIds.size() < 8` 조건 검증 단위 테스트 |
| B-7 | 마지막 멤버(1명)만 남은 그룹에서 탈퇴 | 나 혼자 남은 그룹에서 탈퇴 | 소유자 = 유일 멤버. 탈퇴 시 그룹 자동 해체 여부 미정. 영상/채팅 고아 데이터 생성 가능 | 중간 | 중 | 1인 그룹에서 탈퇴 시나리오. 자동 해체 로직 및 Firestore 문서 정리 확인 |
| B-8 | 그룹 해체 직후 다른 멤버가 그룹 상세 화면에 체류 중 | 소유자가 그룹 삭제, 다른 멤버는 해당 그룹 영상 시청 중 | `groups/{gid}` 문서 삭제 → Firestore `onSnapshot`이 삭제 이벤트 수신 → 화면 강제 이탈 처리 없으면 빈 화면 또는 크래시 | 낮음 | 상 | 그룹 삭제 이벤트 수신 시 안전 이탈 + 토스트 구현 E2E |

### C. 영상 촬영 (capture)

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| C-1 | 카메라 권한 거부 후 재진입 | 첫 권한 요청 거부 → 그룹 상세 "지금 촬영" 재탭 | 권한 재요청이 iOS에서 불가(설정 이동 필요). 안내 화면 없이 그냥 카메라 미리보기가 검은 화면으로 표시될 수 있음 | 높음 | 중 | iOS/Android 각각 권한 거부 → 재진입 시나리오 기기 테스트. `expo-camera` 권한 상태 체크 후 안내 화면 분기 확인 |
| C-2 | 촬영 중 저장공간 부족 | 기기 남은 공간 < 임시 영상 크기(~1 MB) | `expo-camera` 녹화 실패 오류 발생. 오류 처리 없으면 3초 후 빈 파일이 업로드 큐에 등록될 수 있음 | 중간 | 중 | 기기 저장공간을 의도적으로 가득 채운 후 촬영 시도. 오류 메시지 및 큐 등록 차단 확인 |
| C-3 | 3초 카운트다운 중 전화 수신 | 카운트다운 1초 남았을 때 전화 착신 | iOS: 자동으로 카메라 세션 중단 → 통화 종료 후 앱 포그라운드 복귀 시 카운트다운 상태가 reset 안 되면 영상 0초 녹화됨 | 높음 | 중 | 실기기에서 카운트다운 중 전화 수신 테스트. `AppState` 변경 감지 후 촬영 취소/재시작 로직 확인 |
| C-4 | 3초 카운트다운 중 백그라운드 전환 | 홈 버튼/스와이프로 앱 백그라운드 이동 | iOS는 카메라 사용 앱 백그라운드 진입 시 카메라 세션 자동 중단. 포그라운드 복귀 시 세션 재개 필요. 카운트다운 중이면 상태 초기화 필요 | 높음 | 중 | `AppState` 리스너 단위 테스트. 백그라운드 → 포그라운드 복귀 시 카운트다운 초기화 확인 |
| C-5 | HEIC/HEVC 형식으로 촬영 | iOS 기본 설정이 HEVC인 기기에서 촬영 | `expo-camera`가 기기 기본 코덱을 따를 경우 HEVC 출력 가능. Android 구버전에서 HEVC 재생 불가 (API 21 미만). H.264 권고했으나 클라이언트 강제 설정 필요 | 중간 | 상 | iOS에서 HEVC 코덱으로 녹화된 파일이 Android 최소 지원 버전에서 재생되는지 확인. `expo-camera`의 `VideoCodec` 옵션 강제 설정 단위 테스트 |
| C-6 | 슬롯 마감(정각) 직전 5초에 촬영 시작 | 09:59:55에 셔터 탭 → 카운트다운 3초 → 10:00:01에 3초 녹화 완료 | 녹화는 09:59:58~10:00:01에 걸침. 업로드 요청 서버 수신 시점이 10:00:02이면 서버는 10:00 슬롯으로 판정 → 09:00 슬롯에는 빈 상태. 사용자는 "올렸는데 없어요" 혼란 | 중간 | 상 | Emulator에서 정각 -5초에 촬영 트리거. 클라이언트 슬롯 키 vs 서버 `request.time` 기반 슬롯 키 비교. grace window 적용 여부 확인 |
| C-7 | 동일 슬롯에 영상 재촬영 후 업로드 (덮어쓰기) | "다시 찍기" 탭 후 재촬영 + "올리기" | Storage 경로 `{groupId}/{slotId}/{uid}.mp4` 고정 → 덮어쓰기 발생. Firestore `posts` 문서 중복 생성 방지 필요. 덮어쓰기 정책이 명시적으로 upsert인지 확인 필요 | 높음(재촬영 기능 사용 시마다) | 중 | 동일 슬롯 2회 업로드 후 Firestore 문서 개수 및 Storage 파일 개수 확인. 1개만 존재해야 함 |
| C-8 | 촬영 중 마이크 권한 회수 (Android) | Android 설정에서 런타임 권한 회수 가능 | Android 13+는 런타임 중 권한 회수 가능. 마이크 없는 상태로 3초 녹화 완료 → 소리 없는 영상 업로드. 사용자 안내 없음 | 낮음 | 중 | Android 13 에뮬레이터에서 촬영 중 마이크 권한 회수. 오디오 트랙 유무 확인 및 사용자 안내 |

### D. 영상 업로드

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| D-1 | 정각 마감 직전 업로드 레이스 (서버-클라이언트 시간 오차) | 클라이언트 시계 09:59:45이지만 서버는 10:00:05 | 클라이언트는 업로드 허용, 서버 Security Rule에서 `request.time >= slotStart + 1hour` → 거부. 사용자는 "올렸는데 왜 안 보여요?" 혼란 | 중간 | 상 | Firebase Emulator에서 클라이언트 시간 조작 후 업로드 시도. Security Rule `request.time` 기준 검증 로직 단위 테스트 |
| D-2 | 업로드 중 네트워크 끊김 | Wi-Fi → 모바일 데이터 전환, 터널/지하철 진입 | `uploadBytesResumable` 세션이 만료되기 전(보통 1주일) 재개 가능. 그러나 앱이 background kill되면 `expo-task-manager`가 재개해야 함. 재개 실패 시 큐에 `failed` 상태로 영구 잔류 | 높음 | 상 | 업로드 중 비행기 모드 → 30초 대기 → 해제. 자동 재개 및 최종 완료 확인. 슬롯 마감 후 큐 제거 확인 |
| D-3 | 동일 슬롯 중복 업로드 시도 (큐 중복 등록) | 네트워크 오류 후 재시도 로직이 큐에 같은 항목을 두 번 추가 | Storage 덮어쓰기는 멱등하지만 Firestore 문서가 중복 생성될 수 있음. `posts` 문서 2개가 같은 `{authorId, hourSlot}` 조합으로 존재하면 그룹 상세 UI에서 2개 영상 표시 | 낮음 | 중 | 큐 재시도 로직에 멱등성 키 검증 단위 테스트. Firestore `setDoc`(upsert) vs `addDoc`(중복) 방식 확인 |
| D-4 | 업로드 큐 무한 재시도 (서버 5xx 오류) | Firebase Storage 서버 장애 시 지수 백오프 재시도 | 슬롯 마감 시간(정각)이 지났는데도 큐에서 계속 재시도하면 배터리/네트워크 낭비. 마감 시간 초과 시 즉시 큐에서 제거해야 함 | 낮음(서버 장애 시) | 중 | Emulator에서 Storage 업로드를 의도적으로 실패시킨 후 지수 백오프 동작 확인. 정각 초과 시 큐 항목 자동 제거 확인 |
| D-5 | 큐 영속화 실패 (AsyncStorage write 실패) | 기기 저장공간 부족, AsyncStorage 손상 | 큐 상태가 메모리에만 있을 때 앱이 강제 종료되면 업로드 항목 유실. 재시작 후 큐가 비어있어 영상이 영구 소실 | 낮음 | 상 | AsyncStorage write 실패 모킹 단위 테스트. 큐 항목 유실 시 사용자 알림 로직 확인 |
| D-6 | 슬롯 마감(다음 정각) 직전 업로드 시작, 마감 후 완료 | 09:59:50에 업로드 시작, 네트워크가 느려서 10:00:10에 완료 | 서버 Security Rule이 업로드 완료 시점(`request.time`) 기준이면 거부. grace 30초가 있으면 10:00:30까지 허용. 정확한 판정 기준 명시 필요 | 중간 | 상 | grace 30초 정책 기반. Emulator에서 의도적 지연 업로드 테스트. grace 내/외 정확한 허용/거부 확인 |
| D-7 | 앱 완전 종료 후 재시작 시 큐 복원 | 업로드 큐 등록 → 스와이프 앱 종료 → 재시작 | `expo-task-manager` background fetch가 iOS에서 OS 재량으로 실행. 앱 재시작 전까지 업로드 미진행 가능. 재시작 후 AsyncStorage에서 큐 복원 → 재개 여부 확인 | 높음 | 상 | 실기기 테스트 필수 |

### E. 그룹 채팅

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| E-1 | 메시지 순서 역전 | 네트워크 지연 차이로 두 메시지가 다른 순서로 도착 | Firestore `serverTimestamp()`를 사용하지 않고 클라이언트 타임스탬프로 정렬하면 시계 오차 있는 기기에서 순서 역전 | 중간 | 중 | 두 기기에서 시계를 1분 차이로 설정 후 동시 메시지 발송. `serverTimestamp` 기반 정렬 확인 |
| E-2 | 오프라인 상태에서 메시지 발송 | 비행기 모드 중 채팅 입력 후 "전송" | Firestore 오프라인 캐시가 메시지를 로컬 저장 후 온라인 복귀 시 자동 동기화. UI에서 "전송 중" 인디케이터 표시 필요 | 높음 | 중 | 비행기 모드 → 메시지 10개 입력 → 온라인 복귀. 모든 메시지 순서대로 전송 확인 |
| E-3 | 추방(강퇴)된 직후 메시지 전송 시도 | 소유자가 멤버 강퇴 → 강퇴된 사용자가 아직 채팅 화면에 있어 메시지 입력 | Firestore Security Rule이 `memberIds array-contains` 체크를 한다면 강퇴 즉시 거부. 그러나 클라이언트의 onSnapshot이 그룹 문서 변경을 인지하고 화면을 이탈시키기까지 수 초의 지연이 있음 | 낮음 | 상 | Emulator에서 강퇴 + 즉시 메시지 발송 동시 시뮬레이션. Security Rule 단위 테스트 + E2E |
| E-4 | 차단된 사용자의 메시지 표시 | 사용자 B가 A를 차단한 후 A가 메시지 발송 | 차단 정책 미정. 차단 구현 전까지는 차단자에게도 피차단자 메시지가 표시됨. 차단 구현 시 기존 메시지 필터링 방식 결정 필요 (클라이언트 필터 vs Firestore 쿼리 필터) | 중간(차단 구현 후) | 중 | 차단 기능 구현 후 차단 → 메시지 발송 → 차단자 UI 확인. 기존 메시지 소급 가림 여부 |
| E-5 | 메시지 삭제 후 타인이 "모두 읽음" 판정 중 | A가 메시지 삭제 → B가 "모두 읽음" 체크 타임스탬프 갱신 중 | 삭제된 메시지는 "삭제된 메시지" placeholder로 자리 유지 (spec 확정). 읽음 판정이 placeholder를 읽은 것으로 처리하는지 확인 | 낮음 | 낮음 | 삭제 + 읽음 표시 동시 시나리오 단위 테스트 |
| E-6 | 메시지 삭제 시간 제한 초과 후 삭제 시도 | (정책 미정) 5분 후 삭제 버튼 비활성화 경계에서 탭 | 클라이언트에서 버튼을 숨겨도 Security Rule이 없으면 직접 Firestore write로 삭제 가능. Rule에서 시간 기반 삭제 허용 범위 설정 필요 | 낮음(악의) | 중 | 삭제 시간 제한 확정 후 Security Rule 단위 테스트. 클라이언트 UI 비활성화 + 서버 Rule 이중 검증 |
| E-7 | 멱등성 — 메시지 발송 실패 후 재전송 | Firestore write 오류 → 재전송 버튼 탭 | 재전송이 `addDoc`을 다시 호출하면 중복 메시지 생성. 클라이언트 임시 ID(메시지 로컬 ID)로 중복 제거 로직 필요 | 중간 | 중 | 메시지 write를 Emulator에서 강제 실패 → 재전송 → Firestore 문서 개수 확인. 1개만 존재해야 함 |

### F. 알림 (FCM)

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| F-1 | 5,000명 동시 알림 수신 폭주 | 매시 정각 Cloud Function 트리거 | thundering herd. Cloud Function cold start + FCM API 집중 호출 + 1,800 그룹 토픽 발송. 처리 완료에 수 분 소요 가능 | 확실(매시간) | 상 | k6로 1,800 토픽 FCM 발송 요청 동시 시뮬레이션. Cloud Function 실행 시간 및 오류율 측정 |
| F-2 | 조용한 시간(방해금지) 중 알림 발송 | 사용자 설정 22:00~07:00 중 알림 발송 | Cloud Function이 발송 전 `users/{uid}.quietHoursStart/End` 체크를 안 하면 설정 무시하고 발송. `quietHours`가 미정이므로 현재는 아예 없는 상태 | 확실(설정 구현 후) | 중 | 조용한 시간 설정 후 해당 시간에 매뉴얼 FCM 발송. 수신 차단 확인 |
| F-3 | iOS 집중 모드(Focus) 활성 중 알림 | 사용자가 수면/업무 집중 모드 설정 | iOS Focus가 알림을 차단하면 FCM 전달은 성공하지만 사용자에게 표시되지 않음. 앱 재진입 시 배지 카운트와 실제 미확인 알림 수 불일치 가능 | 높음 | 낮음 | iOS Focus 모드 활성 후 FCM 발송. 배지 카운트 동기화 확인 |
| F-4 | 토픽 구독 누락 (그룹 가입 후 FCM 토픽 구독 실패) | 그룹 가입 처리 중 FCM 토픽 구독 Cloud Function 오류 | 가입은 됐는데 영상 알림을 받지 못함. 사용자가 인지하기 어려운 silent failure | 낮음 | 중 | 그룹 가입 → FCM 토픽 구독 → Emulator에서 토픽 메시지 발송 → 수신 확인. 구독 실패 재시도 로직 확인 |
| F-5 | 추방(강퇴) 후 FCM 토픽 구독 미해제 | 강퇴 처리 → 토픽 구독 해제 Cloud Function 오류 | 강퇴된 멤버가 계속 그룹 알림을 받음. 프라이버시 침해 + 불필요한 알림 | 낮음 | 상 | 강퇴 → 토픽 구독 해제 확인. 해제 후 FCM 발송 → 강퇴자 미수신 확인 |
| F-6 | 알림 탭 딥링크 동작 (앱 완전 종료 상태) | 알림 탭 시 앱 cold start + 특정 그룹 딥링크 | 앱이 완전 종료된 상태에서 FCM 알림 탭 → cold start → 딥링크 파라미터(`groupId`, `slot`) 복원 실패 시 메인으로만 이동 | 높음 | 중 | iOS/Android 각각 앱 완전 종료 후 FCM 알림 탭. 딥링크 라우팅 정확도 확인 |
| F-7 | 그룹별 알림 ON/OFF 설정 (정책 미정) | 특정 그룹만 알림 끄기 | 그룹별 설정 미정 상태. 구현 시 FCM 조건부 토픽 또는 클라이언트 필터 방식에 따라 테스트 전략 달라짐 | - | 중 | 정책 확정 후 테스트 설계 |

### G. 시간 동기화

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| G-1 | 클라이언트 시계 오차 (기기 시간 수동 조작) | 사용자가 기기 시간을 10분 앞으로 설정 | 클라이언트가 로컬 시간 기반으로 슬롯 키를 생성하면 서버와 불일치. 업로드 허용/거부 판정 오류. 의도적 조작으로 마감 이후 업로드 가능 | 낮음(의도적) ~ 높음(NTP 미동기) | 상 | 기기 시간을 ±10분 조작 후 업로드 시도. Security Rule `request.time` 기준 판정이 클라이언트 시간과 독립적으로 동작하는지 Emulator 테스트 |
| G-2 | DST Spring Forward (시간 건너뜀) | 미국 3월 둘째 일요일 02:00 → 03:00로 점프 | UTC 기반 슬롯 키면 문제없으나, 로컬 표시 계층에서 02:00 슬롯이 사라진 것처럼 보임. 사용자가 "이 시간에 왜 슬롯이 없어요?" 혼란 | 낮음(DST 지역 한정) | 중 | `date-fns-tz`를 사용한 DST 변환 단위 테스트. Spring Forward 경계(01:59:59 → 03:00:00) UTC 슬롯 표시 확인 |
| G-3 | DST Fall Back (시간 중복) | 미국 11월 첫 일요일 02:00 → 01:00로 반복 | 로컬 02:00 슬롯이 UTC로는 2개 존재. UTC 기반이면 두 슬롯이 모두 존재하나 로컬 표시에서 "02:00 (1차)", "02:00 (2차)" 구분 필요 | 낮음(DST 지역 한정) | 중 | Fall Back 경계 UTC 슬롯 2개를 로컬로 표시하는 `toHourSlot` + `formatRelative` 단위 테스트 |
| G-4 | 비행기 모드 복귀 후 시간 동기화 지연 | 비행기 모드 2시간 → 해제 | NTP 재동기화 전 기기 시계가 drift되면 클라이언트 슬롯 키가 서버와 불일치. 복귀 직후 업로드 시도 시 판정 오류 가능 | 중간 | 중 | 비행기 모드 30분 → 해제 → 즉시 업로드 시도. 서버 `request.time`과 클라이언트 슬롯 키 비교 로그 확인 |
| G-5 | 해외여행 timezone 이동 (서울 → LA) | 사용자가 비행기 탑승 후 LA 도착, 기기 timezone 자동 변경 | UTC 기반 슬롯 키면 데이터 문제 없으나, 로컬 표시가 16시간 뒤로 이동. 같은 그룹의 한국 멤버는 밤 11시 슬롯인데 LA 멤버에게는 오전 6시로 표시 | 낮음 | 중 | 기기 timezone 변경 후 그룹 상세 진입. 슬롯 시간 표시가 새 timezone으로 올바르게 변환되는지 확인 |
| G-6 | 서버-클라이언트 시간 delta 누적 | 장기간 앱 사용 중 NTP 미동기 | 앱 시작 시 `serverTimestamp()` delta를 계산해 보정하지 않으면 장기 사용자는 최대 몇 분의 오차 누적 가능 | 낮음 | 중 | 앱 시작 시 delta 계산 로직 단위 테스트. delta > 30초일 때 사용자 경고 표시 확인 |

### H. 신고/차단/콘텐츠

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| H-1 | 음란물 영상 업로드 | 악의적 사용자가 성인 콘텐츠 3초 영상 촬영/업로드 | 자동 모더레이션(Cloud Vision API 등) 미구현 상태. 다른 멤버 8명이 즉시 자동 재생으로 노출됨. Apple 심사 리젝 사유이기도 함 | 낮음(베타) ~ 중간(상용) | 상 | 신고 버튼 구현 여부 및 신고 → 콘텐츠 가림 동작 확인. 자동 모더레이션 API 연동 전 수동 검토 큐 존재 여부 |
| H-2 | 채팅 비속어/혐오 표현 | 그룹 채팅에서 비속어 메시지 전송 | 비속어 필터 미구현 시 실시간 노출. "삭제된 메시지" 처리가 사후 조치이므로 노출 방지 불가 | 중간 | 중 | 비속어 필터 구현 후 테스트. 미구현 시 신고 → 관리자 삭제 플로우 E2E |
| H-3 | 미성년자 사용자 온보딩 완료 | 생년월일 입력 없이 온보딩 통과 | 생년월일 수집 정책 미정. KISA 가이드라인상 만 14세 미만은 법적 보호 필요. 수집 없으면 미성년자 무제한 가입 | 높음(생년월일 미수집 시) | 상 | 온보딩 생년월일 입력 필드 구현 후 미성년자 연령 입력 시 가입 차단 확인 |
| H-4 | 허위 신고 어뷰징 | 악의적 사용자가 특정 멤버의 정상 영상을 반복 신고 | 신고 즉시 가림 정책 채택 시 정상 콘텐츠가 오가림됨. 허위 신고 패널티 정책 필요 | 낮음 | 중 | 동일 콘텐츠 다수 신고 시나리오 테스트. 신고 횟수 임계값 및 자동 가림 vs 검토 대기 분기 확인 |
| H-5 | 신고 즉시 가림 정책 검증 | 신고 버튼 탭 → 해당 영상/메시지 즉시 가림 | 신고자에게만 가려야 하는지, 그룹 전체에 가려야 하는지 정책 미정. 신고자만 가리면 다른 멤버는 계속 노출 | - | 상 | 정책 확정 후 테스트. 신고자 화면 vs 비신고자 화면 동시 확인 |

### I. 개인정보/탈퇴

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| I-1 | 계정 삭제 후 Firestore/Storage 잔여 데이터 | 프로필 → 계정 삭제 완료 | `users/{uid}` 문서, `memberships/{uid}_*` 문서, Storage 영상, 채팅 메시지 작성자 정보가 삭제되지 않고 남음. Apple 4.5.4 위반 | 확실(삭제 기능 구현 후) | 상 | 계정 삭제 후 Firestore/Storage에 uid 관련 문서/파일이 모두 제거됐는지 Emulator에서 확인 |
| I-2 | GDPR 데이터 내보내기 요청 | 유럽 사용자가 "내 데이터 내보내기" 요청 | 기능 미정. EU 거주 사용자가 있으면 GDPR Article 20 준수 필요. 영상 다운로드 포함 여부 불명확 | 낮음(베타 초기) | 중 | 데이터 내보내기 기능 구현 후 사용자 데이터 완전성 확인 (전화번호, 닉네임, 업로드 영상 목록) |
| I-3 | 미성년자 계정 부모 동의 없이 생성 | 온보딩에서 생년월일 미수집 시 | COPPA(미국 13세 미만), KISA(한국 14세 미만) 위반. 글로벌 출시 시 법적 리스크 | 중간(글로벌 출시 시) | 상 | 생년월일 입력 구현 후 연령 기반 가입 차단 E2E. 미성년자 부모 동의 흐름 분기 확인 |
| I-4 | 소유 그룹 위임 실패 후 계정 삭제 진행 | 소유 그룹이 있는 상태에서 바로 계정 삭제 탭 | 소유권 이전 안내 없이 삭제 진행되면 그룹이 ownerId 없는 orphan 상태. 나머지 멤버가 그룹 관리 불가 | 확실(소유 그룹 있는 사용자) | 상 | 소유 그룹 있는 사용자의 계정 삭제 시나리오. 소유권 이전 강제 또는 그룹 해체 선택 UI → 이후 삭제 진행 확인 |
| I-5 | 탈퇴 후 30일 유예 기간 중 재가입 | (정책 미정) 유예 기간 내 동일 번호로 재인증 | 유예 기간 정책 미정. 재가입 시 이전 데이터 복원 여부, 신규 사용자 처리 여부 불명확 | 낮음 | 중 | 탈퇴 유예 정책 확정 후 재가입 시나리오 테스트. 이전 UID와 신규 UID 관계 확인 |

### J. 권한/디바이스

| # | 시나리오 | 트리거 | 예상 사이드 이팩트 | 발생 확률 | 심각도 | 검증 방법 |
|---|---|---|---|---|---|---|
| J-1 | 카메라 권한 거부 후 설정에서 허용 | 권한 거부 → 설정에서 허용 → 앱 포그라운드 복귀 | `AppState`가 `active`로 변경될 때 권한 재체크 로직 없으면 여전히 권한 없음으로 판단하고 촬영 불가 화면 표시 | 높음 | 중 | iOS/Android에서 권한 거부 → 설정 허용 → 포그라운드 복귀. 자동 권한 재체크 확인 |
| J-2 | 알림 권한 미허용 시 안내 흐름 | 앱 첫 실행 알림 권한 거부 | 알림 권한 없으면 FCM 토큰 발급은 되지만 visible push 수신 불가. 권한 안내 화면 없으면 사용자가 알림을 못 받는 이유를 모름 | 높음 | 중 | 알림 권한 거부 시나리오. 권한 안내 bottom sheet 표시 및 "설정 열기" 동작 확인 |
| J-3 | 연락처 권한 거부 후 그룹 초대 | group-create에서 연락처 선택 탭 → 권한 거부 | 연락처 기반 초대 불가 → 초대 링크 공유로 대체 안내 필요. 권한 거부 후 앱 크래시 방지 | 높음 | 낮음 | 연락처 권한 거부 → 초대 화면 대체 UI 표시 확인 |
| J-4 | 다중 디바이스 동일 계정 (iPhone + Android 동시 로그인) | 기기 A에서 로그인 상태로 기기 B에서도 로그인 | Firebase는 다중 기기 동시 로그인 허용. 두 기기에서 같은 슬롯에 동시 업로드 시 Storage 경로 충돌(마지막 쓰기 승리) | 낮음 | 중 | 두 기기 동시 로그인 → 동일 슬롯 동시 업로드 시도. 최종 Storage 파일 1개, Firestore 문서 1개 확인 |
| J-5 | 기기 변경 (구 기기 폐기 후 신 기기 로그인) | 구 기기를 폐기하고 신 기기에서 동일 번호로 인증 | FCM 토큰이 기기 바인딩이므로 구 기기 토큰은 무효화. Cloud Function이 토큰 갱신을 처리하지 않으면 구 토큰으로 FCM 발송 시도 → 오류 누적 | 높음(자연 발생) | 중 | 기기 변경 시나리오. 구 FCM 토큰 무효화 및 신 토큰 등록 확인. `messaging().onTokenRefresh` 처리 확인 |
| J-6 | 저사양 기기에서 8명 영상 자동 재생 | 그룹 상세 진입 시 멤버 8명 영상 동시 자동 재생 | 저사양 기기 하드웨어 디코더 동시 8채널 처리 한계. 메모리 부족 → 앱 강제 종료 | 중간(저사양 기기) | 상 | 저사양 기기(RAM 2GB Android) 또는 에뮬레이터에서 그룹 상세 진입. 영상 로드 실패율 및 메모리 사용량 모니터링 |
| J-7 | 위치 권한 요청 없음 확인 | 앱 어떤 화면에서도 위치 권한을 요청하지 않아야 함 | 위치 정보는 ChimeMe 기능과 무관. 불필요한 권한 요청은 앱스토어 심사 거부 및 사용자 신뢰 저하 원인 | 낮음 | 낮음 | 전체 앱 플로우 테스트 중 위치 권한 요청 팝업 미발생 확인 |

---

## §2 5,000명 동시 부하 시나리오

### 시나리오 1 — 매시 정각 업로드 폭주
**전제조건**: 5,000 DAU 중 활성 사용자 60% = 3,000명. 3개 그룹 가입 기준 9,000개 그룹-사용자 조합. 정각 이후 3분 이내 업로드율 30% = 2,700건 동시 업로드.

**예상 이벤트 체인**: Storage `uploadBytesResumable` 2,700건 → Cloud Function 2,700개 동시 실행(cold start) → Firestore 쓰기 24,300회/분 burst → FCM 1,800 토픽 API 호출.

**측정 지표**: Firestore 쓰기 오류율 < 1%, 평균 업로드-to-visible 시간 < 15초, FCM 수신율 > 95%.

### 시나리오 2 — 그룹 상세 동시 진입 (영상 재생 부하)
**전제조건**: FCM 알림 발송 직후 30초 이내 알림 탭으로 그룹 상세 진입 50% = 2,500명 동시.

**예상 이벤트 체인**: `posts` 쿼리 2,500건 동시 → Storage 영상 20,000건 origin 요청(CDN 미적용 시) → 순간 20 GB 전송 → `unreadVideoCount=0` 쓰기 2,500건.

**측정 지표**: Storage P95 다운로드 < 3초, Firestore 읽기 오류율 < 0.1%, 기기 메모리 < 300 MB.

### 시나리오 3 — 그룹 채팅 동시 메시지 발송
**전제조건**: 채팅 활성 2,000명 × 3 그룹 = 6,000 동시 리스너, 평균 3명 동시 타이핑.

**예상 이벤트 체인**: `onSnapshot` 6,000개 유지 → 메시지 5,400 write/분 → fan-out 이벤트 전달 → `memberships` 8개 읽기 × 5,400 = 43,200회/분.

**측정 지표**: P95 메시지 수신 < 500ms, 리스너 연결 오류율 < 0.5%.

### 시나리오 4 — 베타 오픈 직후 신규 가입 폭주
**전제조건**: 24시간 1,000명, 첫 1시간 300명.

**예상 이벤트 체인**: SMS 300건/시간 (무료 한도 3% 즉시 소진) → `users` 문서 300건 동시 생성 → FCM 토큰 등록 300건.

**측정 지표**: SMS 발송 성공률 > 99%, OTP 검증 P95 < 3초.

---

## §3 사이드 이팩트 우선순위 매트릭스

| 우선순위 | 항목 | 영역 | 발생 조건 | 영향 범위 | 완화 전략 |
|---|---|---|---|---|---|
| **P0** | 슬롯 키 timezone 불일치로 다른 시간대 멤버 간 영상 분리 | 시간 동기화 | 다른 timezone 멤버 공존 + 로컬 시간 기반 슬롯 키 저장 | 그룹 전체 핵심 기능 무력화 | UTC 저장 + 로컬 표시 원칙 확정 및 Security Rule 적용 |
| **P0** | Firestore 정각 쓰기 폭주 (lastActivityAt + unreadVideoCount fan-out) | Firestore | 5,000 DAU 정각 동시 업로드 | Firestore 1초 1문서 쓰기 제한 충돌 → 업로드 실패 cascade | Cloud Function 비동기 배치 갱신 + lastReadAt 기반 클라이언트 계산 전환 |
| **P0** | SMS toll fraud (App Check 미적용) | 인증 | 앱 출시 후 봇 공격 | SMS 비용 무제한 폭탄 | App Check (DeviceCheck/PlayIntegrity) + IP/번호 rate limit Day-1 적용 |
| **P0** | 앱스토어 리젝 — UGC 신고/차단 미구현 | 콘텐츠 | MVP 출시 | Apple 가이드라인 1.2 위반 → 심사 거부 | 최소 신고 버튼 + 관리자 검토 큐 MVP 포함 |
| **P0** | 계정 삭제 후 데이터 미삭제 | 개인정보 | 계정 삭제 기능 구현 후 | Apple 4.5.4 위반 → 심사 거부. GDPR 위반 | 삭제 시 Firestore + Storage 완전 정리 파이프라인 |
| **P1** | 정각 마감 직전-직후 업로드 경계 레이스 (클럭 스큐) | 업로드 | 디바이스 시간 오차 환경 | 사용자가 올렸는데 안 보이는 신뢰도 저하 | grace 30초 정책 + Security Rule `request.time` 기준 단일화 |
| **P1** | FCM thundering herd (정각 1,800 그룹 동시 발송) | FCM | 매시간 정각 | Cloud Function 과부하 → 알림 지연/누락 | jitter 0~120초 + min instances 2~3 + 토픽 메시징 |
| **P1** | 그룹 소유자 탈퇴 시 orphan 그룹 | 그룹 관리 | 소유자 탈퇴 | 그룹 관리 불가, 멤버 갇힘 | 소유권 이전 강제 또는 그룹 해체 선택 UI |
| **P1** | 추방 후 FCM 토픽 구독 미해제 | FCM + 프라이버시 | 강퇴 처리 실패 | 강퇴자가 계속 그룹 알림 수신 (프라이버시 침해) | 강퇴 처리 시 토픽 구독 해제 트랜잭션 처리 |
| **P1** | 업로드 큐 앱 강제 종료 후 영속화 실패 | 업로드 | 저장공간 부족, AsyncStorage 오류 | 영상 유실 (사용자 콘텐츠 손실) | AsyncStorage write 실패 fallback + 큐 항목 무결성 체크 |
| **P1** | 슬롯 마감 직전 촬영 시작 → 마감 후 업로드 완료 | 업로드 | 정각 -5초 이내 촬영 시도 | 업로드 거부 + 사용자 혼란 | grace window + 클라이언트 마감 임박 UI 경고 |
| **P2** | Storage CDN 미적용 월 $300+ 비용 | Storage 비용 | 5,000 DAU 활발한 사용 | 예산 초과 | Firebase Hosting CDN + 썸네일 우선 로딩 |
| **P2** | SIM 스왑 후 기존 세션 미만료 | 인증 보안 | SIM 스왑 공격 성공 | 계정 탈취 | 새 기기 로그인 시 기존 세션 강제 만료 구현 |
| **P2** | DST Fall Back 중복 슬롯 표시 | 시간 동기화 | DST 지역 사용자 | 슬롯 표시 혼란 | UTC 슬롯 키 + 로컬 표시 계층 DST 처리 |
| **P2** | 저사양 기기 8명 영상 동시 재생 메모리 부족 | 성능 | 저사양 기기 사용자 | 앱 강제 종료 | 지연 로딩, 화면 밖 영상 언로드, 화질 적응형 스트리밍 |
| **P3** | OTP 코드 만료 후 사용자 안내 부재 | 인증 UX | SMS 수신 지연 사용자 | 앱이 멈춘 것처럼 보임 | `auth/code-expired` 오류 → 재전송 안내 인라인 표시 |
| **P3** | 그룹 상세 체류 중 그룹 해체 (orphan 화면) | 그룹 관리 | 소유자 해체 + 멤버 체류 | 빈 화면 / 크래시 | `onSnapshot` 삭제 이벤트 → 안전 이탈 + 토스트 |

---

## §4 테스트 전략 권고

### 4.1 테스트 피라미드
| 레이어 | 비중 | 도구 | 대상 |
|---|---|---|---|
| 단위 (Unit) | 70% | Jest + React Native Testing Library | `toHourSlot`, `formatRelative`, Security Rules, 큐 멱등성, 슬롯 마감 판정, DST 변환 |
| 통합 (Integration) | 20% | Firebase Emulator Suite (Auth/Firestore/Storage/Functions) | 업로드 플로우, 그룹 가입 race condition, 채팅 메시지 순서, 계정 삭제 파이프라인 |
| E2E | 10% | Detox (iOS/Android 실기기) | 인증 → 온보딩 → 그룹 생성 → 촬영 → 업로드 → 그룹 상세 확인 전체 happy path |

### 4.2 Firebase Emulator 활용
```bash
firebase emulators:start --only auth,firestore,storage,functions,pubsub
npm install --save-dev @firebase/rules-unit-testing
```

**Security Rules 단위 테스트 (슬롯 마감 경계)**:
```javascript
import { initializeTestEnvironment, assertFails, assertSucceeds } from '@firebase/rules-unit-testing';
import { doc, setDoc } from 'firebase/firestore';

describe('posts 업로드 슬롯 마감 Security Rule', () => {
  let testEnv;
  beforeAll(async () => {
    testEnv = await initializeTestEnvironment({
      projectId: 'chimeme-test',
      firestore: { rules: fs.readFileSync('firestore.rules', 'utf8') },
    });
  });

  it('정각 +29초에 업로드를 허용한다', async () => {
    const db = testEnv.authenticatedContext('user1').firestore();
    await assertSucceeds(
      setDoc(doc(db, 'groups/g1/posts/p1'), {
        authorId: 'user1',
        hourSlot: new Date('2026-05-12T01:00:00.000Z'),
        videoUrl: 'gs://bucket/video.mp4',
      })
    );
  });

  it('정각 +31초에 업로드를 거부한다', async () => {
    const db = testEnv.authenticatedContext('user1').firestore();
    await assertFails(
      setDoc(doc(db, 'groups/g1/posts/p1'), {
        authorId: 'user1',
        hourSlot: new Date('2026-05-12T01:00:00.000Z'),
        videoUrl: 'gs://bucket/video.mp4',
      })
    );
  });

  afterAll(async () => testEnv.cleanup());
});
```

### 4.3 k6 부하 테스트 (정각 thundering herd)
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  scenarios: {
    hourly_burst: {
      executor: 'ramping-arrival-rate',
      startRate: 0,
      timeUnit: '1s',
      preAllocatedVUs: 500,
      stages: [
        { duration: '10s', target: 300 },
        { duration: '50s', target: 50 },
        { duration: '60s', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<10000'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const groupId = `group_${Math.floor(Math.random() * 1800)}`;
  const userId = `user_${__VU}`;
  const slotKey = '2026051210';
  const uploadUrl = `https://firebasestorage.googleapis.com/v0/b/chimeme-test.appspot.com/o/${groupId}%2F${slotKey}%2F${userId}.mp4`;
  const res = http.post(uploadUrl, open('./fixtures/sample_3sec.mp4', 'b'), {
    headers: { 'Content-Type': 'video/mp4', 'Authorization': `Bearer ${__ENV.FIREBASE_TOKEN}` },
  });
  check(res, {
    '업로드 성공': (r) => r.status === 200,
    'P95 10초 이내': (r) => r.timings.duration < 10000,
  });
}
```

### 4.4 그룹 8명 초과 race condition 통합 테스트
```javascript
import { initializeTestEnvironment } from '@firebase/rules-unit-testing';
import { doc, runTransaction, arrayUnion } from 'firebase/firestore';

it('7명 꽉 찬 그룹에 동시 2명 가입 시도 시 1명만 가입된다', async () => {
  const testEnv = await initializeTestEnvironment({ projectId: 'chimeme-test' });
  const adminDb = testEnv.withSecurityRulesDisabled().firestore();
  await adminDb.doc('groups/g1').set({
    memberIds: ['u1','u2','u3','u4','u5','u6','u7'], maxMembers: 8,
  });
  const db8 = testEnv.authenticatedContext('u8').firestore();
  const db9 = testEnv.authenticatedContext('u9').firestore();
  const results = await Promise.allSettled([
    runTransaction(db8, async (tx) => {
      const snap = await tx.get(doc(db8, 'groups/g1'));
      if (snap.data().memberIds.length >= 8) throw new Error('FULL');
      tx.update(doc(db8, 'groups/g1'), { memberIds: arrayUnion('u8') });
    }),
    runTransaction(db9, async (tx) => {
      const snap = await tx.get(doc(db9, 'groups/g1'));
      if (snap.data().memberIds.length >= 8) throw new Error('FULL');
      tx.update(doc(db9, 'groups/g1'), { memberIds: arrayUnion('u9') });
    }),
  ]);
  const finalSnap = await adminDb.doc('groups/g1').get();
  expect(finalSnap.data().memberIds.length).toBe(8);
  expect(results.filter(r => r.status === 'fulfilled').length).toBe(1);
  await testEnv.cleanup();
});
```

---

## §5 개발자에게 보강 요청 (DR-1~DR-5)

| # | 항목 | 배경 | 우선순위 |
|---|---|---|---|
| **DR-1** | Security Rule에서 slotStart 시간 범위 검증 로직 명시 (grace 30초 포함) | `spec/group-detail/data.md:43` "소급 업로드 금지" 명시만 있고 Rule 초안 없음 | P0 |
| **DR-2** | expo-task-manager 큐 영속화 스키마 (AsyncStorage key 구조, 상태 enum) 공유 | 스키마 없이 유닛 테스트 목 구조 불일치 | P0 |
| **DR-3** | FCM 토픽 구독/해제 Cloud Function 코드 + 실패 핸들링 공유 | F-4/F-5 테스트 위해 인터페이스 필요 | P1 |
| **DR-4** | 채팅 메시지 멱등성 키(로컬 임시 ID) 구현 방식 공유 | E-7 중복 메시지 방지 테스트 설계 차단 | P1 |
| **DR-5** | HEVC → H.264 강제 변환 코드 위치 명시 | C-5 코덱 강제 설정 단위 테스트 작성 불가 | P1 |

---

## §6 기획자에게 결정 요청 (PD-1~PD-5)

| # | 결정 항목 | QA 임팩트 | 우선순위 |
|---|---|---|---|
| **PD-1** | 신고/차단 정책 확정 (즉시 가림 vs 검토 대기 / 차단 시 기존 메시지 소급 / 허위 신고 패널티) | UGC 안전 테스트 전체 차단. Apple 1.2 심사 직결 | P0 |
| **PD-2** | 메시지 삭제 시간 제한 확정 | Security Rule 시간 기반 삭제 허용 조건 작성 불가 | P1 |
| **PD-3** | 그룹 탈퇴/강퇴/해체 시 콘텐츠 처리 정책 확정 (탈퇴자 영상 잔존 / 강퇴자 메시지 표시 / 소유자 자동 이전 vs 해체) | 그룹 라이프사이클 통합 테스트 차단 | P1 |
| **PD-4** | 본인 영상 삭제/교체 정책 확정 (교체 횟수 / 삭제 후 슬롯 상태) | C-7 덮어쓰기 검증 기준 미정 | P1 |
| **PD-5** | 채팅 보존 기간 확정 (영구/30일/90일) | 메시지 정리 Cloud Function 테스트 차단 | P2 |

---

## §7 P0 Top 5 요약

| 순위 | 항목 | review-developer.md 언급 여부 | 테스터 추가 관점 |
|---|---|---|---|
| 1 | 슬롯 timezone UTC 저장 미확정 | 언급됨 (Critical #1) | DST Fall Back 중복 슬롯(G-3), Spring Forward(G-2) 단위 테스트, 해외여행 timezone 이동(G-5) 미언급 |
| 2 | 정각 Firestore 쓰기 폭주 | 언급됨 (Critical #2) | §2 시나리오 1에서 24,300회/분 burst 구체적 측정 지표 + k6 재현 방법 제시. 사용자 증상("올렸는데 안 보여요") 누락 |
| 3 | UGC 신고/차단 완전 미구현 ★ | "앱스토어 리젝" 맥락으로만 언급 | 음란물 자동 재생 노출(H-1), 차단 사용자 채팅 필터링(E-4), 허위 신고 어뷰징(H-4) 실제 안전 시나리오 누락. PD-1 결정 선행 필요 |
| 4 | SIM 스왑 후 기존 세션 미만료 ★ | "방어 불가"로 종결 | 새 기기 로그인 시 구 세션 강제 만료 검증(A-5). Firebase 기본동작은 만료 안 함 → 추가 구현 필수 |
| 5 | 계정 삭제 후 잔여 데이터 ★ | review-developer 누락 | Apple 4.5.4 + GDPR 동시 위반. `users`, `memberships`, Storage 영상, 채팅 작성자 정보 각각 삭제 확인(I-1) 필요 |

---

## References

- `spec/main/interactions.md` — 새 영상 판정 + unread 카운트 갱신
- `spec/main/data.md` — Firestore groups 쿼리, memberships, 복합 인덱스
- `spec/group-detail/interactions.md` — 업로드 윈도우 60분, 소급 금지
- `spec/group-detail/data.md:24-26` — hourSlot timezone "사용자 로컬" (UTC 전환 필요)
- `spec/group-detail/data.md:43` — Security Rule 소급 업로드 금지 메모
- `spec/capture/overview.md:21-31` — 업로드 윈도우, 큐, 재시도, 권한 거부 정책
- `spec/chat/overview.md:21-30` — 읽음 표시, 삭제, 영상 인용 정책
- `spec/notifications/overview.md` — 방해금지 시간 설정
- `spec/auth-phone/overview.md` — 전화번호 인증 흐름
- `spec/onboarding/overview.md` — 닉네임, 프로필, 약관 동의
- `spec/profile/overview.md` — 계정 삭제, 소유 그룹 위임
- `docs/2026-05-11/decisions.md:49-74` — 그룹 8명, 60분 윈도우, 7일 보존, 미정 사항
- `docs/2026-05-12/review-developer.md` — 개발자 기술 검토서
- `docs/2026-05-12/review-planner.md` — 기획 리뷰
