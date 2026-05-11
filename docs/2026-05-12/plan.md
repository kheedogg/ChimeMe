# 2026-05-12 작업 계획 — 3인 팀 리뷰 통합

> **목적**: 기획자·개발자·테스터 3인 에이전트 팀의 리뷰를 통합하여 5,000명 동시 사용자 기준 사이드 이팩트 없는 ChimeMe 기획안을 완성하기 위한 실행 계획.

---

## 1. 오늘 진행 사항 요약

- 3인 에이전트 팀 구성 및 병렬 리뷰 완료
  - **기획자** → [`review-planner.md`](./review-planner.md): 화면 완성도 25%, 비즈니스 갭 9영역, 우선순위 P0~P2 체크리스트
  - **개발자** → [`review-developer.md`](./review-developer.md): Firestore/Storage/FCM/Auth 한계, Critical Issue Top 5
  - **테스터** → [`review-tester.md`](./review-tester.md): 엣지케이스 카탈로그(A~J, 60+ 시나리오), P0~P3 매트릭스, k6 부하 시나리오
- ChimeMe 깃 저장소 초기화 + `https://github.com/kheedogg/ChimeMe`에 첫 푸시

---

## 2. 3인 리뷰 교차 검증 — 합의된 핵심 사실

3명 모두 **동일하게 강조한 P0급 이슈** (즉시 결정 필요):

| # | 합의 항목 | 출처 |
|---|---|---|
| C1 | **슬롯 키 시간대 기준** — 현재 `spec/group-detail/data.md:24-26`의 "사용자 로컬" 정의는 그룹 내 다른 timezone 멤버 공존 시 치명적 버그 유발 | planner §2.8, developer §3.1, tester G-1~G-5 |
| C2 | **Firestore 정각 쓰기 폭주** — `lastActivityAt` + `unreadVideoCount` fan-out이 5,000 DAU 정각 동시 업로드 시 1초 1문서 쓰기 제한과 충돌 | developer §1.2 #2-3, tester §2 시나리오 1 |
| C3 | **UGC 신고/차단 미구현** — Apple 가이드라인 1.2 위반 → MVP 출시 시 심사 리젝 확정 | planner §2.2, developer §6.3, tester H-1~H-5 |
| C4 | **SMS toll fraud (App Check 미적용)** — 출시 즉시 봇 공격 시 SMS 비용 무제한 폭탄 | developer §5.2, tester A-4·A-6 |
| C5 | **계정 삭제 후 데이터 잔존** — Apple 4.5.4 위반 + GDPR 위반. 현재 기획서 전무 | planner §2.6, tester I-1 (developer 미언급) |

→ 위 5가지는 **개발 코드 작성 이전에 정책/아키텍처 확정이 선행되어야 함**.

---

## 3. 결정/실행 분리

### 3.1 즉시 적용 (사용자 추가 결정 불필요)
모든 에이전트가 동일한 기술 답을 제시한 항목 — `decisions.md`에 기록 후 spec/ 반영.

| ID | 결정 | 근거 |
|---|---|---|
| D-12-01 | **슬롯 키는 UTC `YYYYMMDDHH` 저장 / 표시만 로컬 변환** | 3명 합의. spec/group-detail/data.md:24-26 수정 필요 |
| D-12-02 | **업로드 grace window 30초** (정각 마감 후 30초까지 허용) | developer §3.2, tester D-6 합의 |
| D-12-03 | **영상 인코딩: 720p H.264 / 30fps / 2~3Mbps / MP4** | developer §2.1, tester C-5 (HEVC 호환성 우려) |
| D-12-04 | **App Check Day-1 적용** (iOS DeviceCheck, Android Play Integrity) | developer §5.3, tester A-6 |
| D-12-05 | **lastActivityAt / unreadVideoCount 쓰기 → lastReadAt 기반 클라이언트 계산으로 전환** | developer §1.2 #2-3, tester P0 |
| D-12-06 | **FCM 토픽 메시징 + jitter 0~120초 + min instances 2~3** | developer §4.2, tester F-1 |
| D-12-07 | **업로드 큐는 `expo-task-manager` + AsyncStorage 영속화, 지수 백오프 5회** | developer §2.5, tester D-2·D-5 |
| D-12-08 | **채팅 페이지네이션 50건/페이지, 메시지 정렬 `serverTimestamp` 기반** | developer §1.4, tester E-1 |
| D-12-09 | **계정 삭제 시 Firestore `users/{uid}` + `memberships/{uid}_*` + Storage 영상 + 채팅 작성자 정보 전부 정리** | tester I-1 |
| D-12-10 | **Storage 자동 삭제 = Cloud Storage Lifecycle `age:8` + Cloud Function 정리 이중 파이프라인** | developer §2.4 |

### 3.2 사용자 결정 필요 (8개 항목)
3인 모두 결정을 요청한 정책 — `decisions.md` 미정란에 명시.

| ID | 항목 | 추천안 | 출처 |
|---|---|---|---|
| Q-12-01 | 1인 최대 그룹 수 | 10개 (성능/비용 균형) | planner, developer §8.1 #2 |
| Q-12-02 | 영상 보존 기간 | 현 7일 유지 | docs/2026-05-11 decisions.md:51 |
| Q-12-03 | 미성년자 정책 (생년월일 수집 / 만 14세 미만 차단) | 한국 우선 출시 시 KISA 준수 → 만 14세 이상 가입 차단 | planner §2.3, tester H-3·I-3 |
| Q-12-04 | UGC 신고/차단 정책 (즉시 가림 vs 검토 / 신고자만 vs 그룹 전체 / 허위 신고 패널티) | 신고자 화면에서만 즉시 가림 + 3건 이상 누적 시 그룹 전체 가림 + 관리자 검토 | tester PD-1 |
| Q-12-05 | 메시지 삭제 시간 제한 | 5분 (Slack 기본값 참고) | tester PD-2 |
| Q-12-06 | 그룹 탈퇴/강퇴/해체 시 콘텐츠 처리 (탈퇴자 영상 잔존 / 강퇴자 메시지 표시 / 소유자 자동 이전) | 탈퇴자 영상 유지, 강퇴자 메시지 placeholder, 소유자 탈퇴 시 그룹 해체 강제 선택 UI | planner §2.1, tester PD-3 |
| Q-12-07 | 본인 영상 삭제/교체 정책 | 슬롯 마감 전까지 무제한 교체, 마감 후 삭제만 가능 | planner §3.7, tester PD-4 |
| Q-12-08 | 채팅 보존 기간 | 90일 (영구 보관 시 Firestore 비용 부담) | tester PD-5 |
| Q-12-09 | 푸시 알림 기본 ON/OFF + 다이제스트 옵션 | 기본 ON, 3개 이상 그룹 가입 시 다이제스트 옵션 노출 | planner §2.9, developer §4.4 |
| Q-12-10 | 그룹 초대 방식 | SMS 딥링크 + QR 코드 (카카오톡 공유는 P2) | planner §2.7, docs/2026-05-11 decisions.md:72 |
| Q-12-11 | 지원 국가 / SMS 화이트리스트 | 베타: 한국 only (+82), 상용 시 단계적 확대 | tester A-2·A-8 |
| Q-12-12 | 약관 동의 시점 (auth-phone vs onboarding vs 분리) | auth-phone 화면 하단 체크박스 + 필수/선택 분리 | planner §2.4 |

---

## 4. 단계별 실행 계획

### Phase 0 — 정책 확정 (오늘~내일)
**목표**: 위 Q-12-01 ~ Q-12-12 12개 항목에 대한 사용자 답변 수령 → `decisions.md` 확정.

체크리스트:
- [ ] 사용자 결정 12개 항목 답변
- [ ] `decisions.md` 결정 사항 반영
- [ ] `spec/group-detail/data.md` UTC 슬롯 정의 수정
- [ ] `spec/main/data.md` 핫 쓰기 회피 모델로 수정 (lastReadAt 기반)
- [ ] 깃 두 번째 푸시

### Phase 1 — 화면 기획 완성 (1~2일)
**목표**: 스텁 상태 8개 화면을 main/group-detail 수준으로 보강 (4문서 구조).

체크리스트:
- [ ] `spec/auth-phone/` ui.md, interactions.md, data.md 작성 (약관 동의 포함)
- [ ] `spec/auth-verify/` ui.md, interactions.md, data.md (재전송 쿨다운, OTP 만료)
- [ ] `spec/onboarding/` ui.md, interactions.md, data.md (생년월일, 약관)
- [ ] `spec/group-create/` ui.md, interactions.md, data.md (초대 방식)
- [ ] `spec/capture/` ui.md, interactions.md, data.md (720p H.264 명시)
- [ ] `spec/chat/` ui.md, interactions.md, data.md (페이지네이션 50건, 삭제 5분)
- [ ] `spec/profile/` ui.md, interactions.md, data.md (계정 삭제, 신고 목록, 차단 목록)
- [ ] `spec/notifications/` ui.md, interactions.md, data.md (방해금지, 그룹별 ON/OFF)

### Phase 2 — 신규 화면 기획 (1일)
**목표**: MVP에 추가 필요한 화면 신규 작성.

체크리스트:
- [ ] `spec/report/` — 영상/메시지/사용자 신고 화면 (4문서)
- [ ] `spec/group-settings/` — 그룹 관리 화면 (멤버 추방, 그룹 해체, 권한)
- [ ] `spec/blocked-users/` — 차단 사용자 목록
- [ ] `spec/account-delete/` — 계정 삭제 흐름 (소유 그룹 처리)
- [ ] `spec/legal/` — 이용약관 / 개인정보처리방침 표시 (또는 외부 URL)

### Phase 3 — 인프라/계정 준비 (병렬 진행)
사용자가 직접 진행해야 하는 영역. 별도 체크리스트는 이전 답변(`Firebase 셋업 / 스토어 계정 / 법적 문서`)에서 정리됨.

체크리스트:
- [ ] Firebase 프로젝트 2개(dev/prod) 생성 + 서울 리전 + Blaze
- [ ] App Check 활성화 (DeviceCheck / Play Integrity)
- [ ] Apple Developer Program / Google Play Console 등록
- [ ] EAS 계정 + 환경변수 분리
- [ ] 이용약관 / 개인정보처리방침 문서 작성 + URL 호스팅

### Phase 4 — 개발 착수 준비 (Phase 0~3 완료 후)
- [ ] Expo 프로젝트 스캐폴딩 (`npx create-expo-app`)
- [ ] Firebase SDK + App Check 통합
- [ ] Firestore Security Rules 초안 작성 (UTC 슬롯 검증 포함)
- [ ] Cloud Functions 초안 (영상 finalize 트리거, FCM 토픽 발송, 7일 정리)
- [ ] Firebase Emulator Suite 로컬 환경 구성
- [ ] k6 부하 테스트 시나리오 구성 (tester §4.3)

---

## 5. 리스크 / 블로커

| 리스크 | 영향 | 완화 |
|---|---|---|
| 사용자 결정 12개 항목 지연 | Phase 1~2 화면 작성 지연 | 추천안을 임시 채택하여 작업 시작, 결정 후 차이만 수정 |
| 이용약관 / 개인정보처리방침 외주 의뢰 시간 | Phase 3 완료 지연 → 앱 심사 제출 불가 | 초안은 Claude로 작성 → 변호사 검토 의뢰 병행 |
| Firebase Blaze 결제 카드 미등록 | Cloud Functions 사용 불가 | Phase 0 단계에서 미리 등록 |
| 5,000명 부하 테스트 환경 비용 | k6 클라우드 + Firebase 비용 | Emulator 기반 우선, 실제 부하 테스트는 베타 직전 1회만 |

---

## 6. 오늘 산출 파일

- `docs/2026-05-12/review-planner.md` ✓
- `docs/2026-05-12/review-developer.md` ✓
- `docs/2026-05-12/review-tester.md` ✓
- `docs/2026-05-12/plan.md` (이 파일) ✓
- `docs/2026-05-12/decisions.md` ✓
- ChimeMe git push #1 ✓

## 7. 다음 작업

1. **사용자**: Q-12-01 ~ Q-12-12 12개 결정 항목 답변
2. **Claude**: 답변 수령 후 `decisions.md` 결정란 채우기 + spec/ 보정 + Phase 1 화면 기획 작업 시작
