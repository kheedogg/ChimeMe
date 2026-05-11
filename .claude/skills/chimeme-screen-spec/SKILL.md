---
name: chimeme-screen-spec
description: ChimeMe 프로젝트의 화면별 기획서(spec/<screen>/) 및 날짜별 실행 기록(docs/YYYY-MM-DD/) 작성 규칙. 신규 화면 추가, 기존 화면 변경, 일일 작업 시작/종료 시 반드시 따라야 한다.
---

# ChimeMe — 화면 기획 & 실행 기록 워크플로

이 스킬은 `/Users/doheekim/home/MyApp/` (ChimeMe 프로젝트) 안에서 작업할 때 적용되는 디렉토리 규칙이다.

## 핵심 규칙

1. **앱 전체 기획서는 화면별로 별도 폴더를 구성한다.**
   - 위치: `spec/<screen-key>/`
   - 각 화면 폴더는 아래 표준 파일을 갖는다:
     - `overview.md` — 화면 목적, 사용자 시나리오, 진입점
     - `ui.md` — UI 구성요소, 레이아웃, 디자인 토큰, 컬러/타이포
     - `interactions.md` — 사용자 인터랙션, 상태 전이, 에러 케이스
     - `data.md` — 사용/생성 데이터, Firestore 쿼리, 캐시 전략
   - 인덱스는 `spec/README.md`에 한 줄씩 등록한다.

2. **화면별 실행(구현·작업) 내용은 `docs/`에 기록한다.**
   - 위치: `docs/YYYY-MM-DD/`
   - 같은 화면을 여러 날 작업하면 `docs/<date>/<screen-key>.md` 형식으로 화면 키를 파일명에 포함시켜 추적 가능하게 한다.
   - 의사결정은 `docs/<date>/decisions.md`, 하루 요약은 `docs/<date>/progress.md`에 모은다.

3. **화면 키 명명 규칙**
   - kebab-case, 영문
   - 현재 정의된 화면 키:
     - `main` — 그룹 리스트 (홈)
     - `group-detail` — 그룹 상세 (시간대별 영상 뷰)
     - `group-create` — 그룹 생성 · 초대
     - `auth-phone` — 전화번호 입력
     - `auth-verify` — SMS 코드 확인
     - `onboarding` — 신규 사용자 닉네임/프로필 입력
     - `capture` — 3초 영상 촬영
     - `chat` — 그룹 채팅
     - `profile` — 내 프로필
     - `notifications` — 알림 설정

## 신규 화면 추가 절차

1. 화면 키를 결정 (위 명명 규칙 준수)
2. `spec/<screen-key>/` 폴더 생성
3. 4개 표준 문서 작성 (`overview.md`, `ui.md`, `interactions.md`, `data.md`)
4. `spec/README.md` 인덱스에 한 줄 추가 — `- [<screen-key>](./<screen-key>/overview.md) — 한 줄 설명`
5. 오늘 작업 진행 내역이 있다면 `docs/<오늘 날짜>/<screen-key>.md`에 작업 기록

## 기존 화면 변경 절차

1. 기획 변경이면 → `spec/<screen-key>/` 안의 해당 문서를 직접 수정
2. 변경 이유·결정은 `docs/<오늘 날짜>/decisions.md`에 누적 기록
3. 구현/리팩터링 작업 진행 시 → `docs/<오늘 날짜>/<screen-key>.md`에 작업 로그

## 일일 작업 시작/종료

- **시작 시**: 오늘 날짜 폴더(`docs/YYYY-MM-DD/`)가 없으면 생성. 작업 계획을 `plan.md`에 적는다.
- **종료 시**: 그날 진행한 화면별 작업을 `progress.md`에 요약. 의사결정은 `decisions.md`에 옮긴다.

## 위반 시 처리
- 화면 기획이 `spec/` 외부(예: `docs/`나 루트)에 흩어져 있으면 → 모아서 `spec/<screen-key>/`로 이동
- 실행 기록이 `spec/` 안에 들어가 있으면 → `docs/<date>/`로 이동
