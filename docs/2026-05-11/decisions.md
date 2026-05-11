# 2026-05-11 — 의사결정 기록

## 확정 사항

### 앱 이름
- **ChimeMe** (확정)
- 이전 후보 "Chime"은 동명 핀테크 서비스(미국 Chime) 존재로 변경
- TODO: 도메인 / 앱스토어 / 상표 사전 검색 (chimeme.app / chimeme.io 등)

### 프로젝트 성격
- **상용 출시 목표** (단, 첫 단계는 제한된 사용자 대상 베타)
- 최소 기능(MVP) 우선 → iOS + Android 동시 빠른 배포

### 백엔드
- **Firebase** 채택
  - Auth, Firestore, Cloud Storage, FCM(푸시) 통합 활용
  - 초기 운영 비용 낮고 인프라 셋업 부담 최소

### 인증
- **전화번호 인증 단일 방식** (확정)
- Firebase Authentication의 Phone Auth 사용
- iOS: APNs 기반 silent push 검증 + (필요 시) reCAPTCHA fallback
- Android: Play Integrity API + SMS 코드
- ⚠ 비용: Firebase Phone Auth SMS는 월 무료 한도 이후 과금 → 베타 사용자 한정으로 시작

### 채팅
- **그룹 채팅만** 지원 (확정)
- DM(1:1) 기능은 추후 검토 대상에서 제외
- Firestore 기반 메시지 구조 (서브컬렉션 또는 별도 컬렉션)

### 클라이언트 플랫폼
- **React Native (Expo)** (확정)
**채택 이유**
- iOS + Android 동시 배포 + 빠른 베타 출시에 가장 적합
- **EAS Build / Submit**: 빌드부터 TestFlight·Play Internal Testing 업로드까지 CLI 한 번
- **EAS Update (OTA)**: 베타 사용자에게 스토어 심사 없이 즉시 패치 배포 가능 → 초기 빠른 반복에 결정적
- `react-native-firebase` 생태계 성숙
- `expo-camera`, `expo-av` 등 짧은 영상 녹화·재생 라이브러리 풍부

**차안: Flutter**
- 성능·UI 일관성은 우수하나, 베타 단계 빠른 OTA 패치 측면에서 Expo가 우위

### 베타 배포 채널
- iOS: **TestFlight** (외부 그룹 최대 10,000명, 초대 링크 가능)
- Android: **Play Console Internal Testing** (이메일/그룹 기반)

## 추가 확정 (Round 1 — 2026-05-11)

- **그룹 최대 인원: 8명**
- **업로드 윈도우: 정각 이후 60분(다음 정각 직전까지)** — 소급 업로드 금지
- **영상 보존 기간: 7일 후 자동 삭제**
- **신규 사용자 프로필 입력: SMS 인증 직후 `onboarding` 전용 화면**

## 추가 확정 (Round 2 — 2026-05-11, 촬영 UX)

- **녹화 트리거: 한 번 탭 → 3-2-1 카운트다운 후 자동 3초 녹화**
- **재촬영: 무제한** (업로드 확정 전까지)
- **업로드: 백그라운드 큐 방식** (`expo-task-manager` + 재시도)
- **카메라 권한 거부 시: 안내 화면 + 설정 이동 버튼**

## 추가 확정 (Round 3 — 2026-05-11, 채팅 정책)

- **메시지 형식**: 텍스트 + 이모지 + 그룹의 이전 영상 인용 (외부 사진/파일 첨부 없음)
- **읽음 표시**: "모두 읽음"만 (멤버 전원 read 시 인디케이터 사라짐). 개별 read status 비공개
- **메시지 삭제/수정**: 본인 메시지 삭제만 가능, 수정 불가. 삭제 시 "삭제된 메시지" placeholder
- **알림**: 영상 알림과 채팅 알림 분리 토글 (`notifications`에서 각각 ON/OFF)

## 미정 (다음 라운드)

- [ ] 그룹 초대 방법, 링크 만료, 그룹 아이콘
- [ ] 푸시 알림 정책 (정각 트리거, 방해 금지 시간, 그룹별 설정)
- [ ] 베타 화이트리스트 운영
- [ ] 한 명이 가질 수 있는 그룹 수 상한
- [ ] 메시지 삭제 시간 제한, 페이지네이션 크기, 영상 인용 탭 동작
