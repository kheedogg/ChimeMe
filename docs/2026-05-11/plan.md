# 2026-05-11 — 프로젝트 킥오프 계획

## 1. 프로젝트 목표

SetLog 앱을 모티브로 한 그룹 일상 공유 앱 개발.

### 핵심 기능
- **그룹 생성/초대**: 친구들끼리 비공개 그룹 형성
- **시간제 알림**: 매 시간 정각(또는 일정 주기) 업로드 알림
- **3초 영상 캡처**: 카메라로 짧은 영상 촬영 후 즉시 업로드
- **타임라인 피드**: 같은 그룹 멤버의 영상을 시간 순서대로 열람

## 2. 결정해야 할 기술 스택 / 방향

| 항목 | 후보 | 비고 |
|---|---|---|
| 플랫폼 | iOS / Android / 크로스플랫폼 | React Native, Flutter, Native |
| 백엔드 | Firebase, Supabase, 자체 서버(Node/Python) | 인증·푸시·미디어 저장 |
| 미디어 저장소 | Firebase Storage, S3, Cloudflare R2 | 비용/성능 |
| 푸시 알림 | FCM / APNS | 시간제 트리거 방식 |
| DB | Firestore, PostgreSQL, MongoDB | 그룹/유저/포스트 모델 |

## 3. 초기 데이터 모델 스케치

```
User       { id, name, profileImage, createdAt }
Group      { id, name, ownerId, memberIds[], createdAt }
Post       { id, groupId, authorId, videoUrl, capturedAt, hourSlot }
Membership { userId, groupId, joinedAt, role }
```

## 4. 다음 액션 (TODO)

- [ ] 기술 스택 결정 (사용자 확인 필요)
- [ ] 와이어프레임 / 화면 흐름 정의
- [ ] 인증 방식 결정 (전화번호 / 이메일 / OAuth)
- [ ] "매 시간 업로드 윈도우" UX 정의 (몇 분간 열려있는지, 놓쳤을 때 처리)
- [ ] 영상 촬영 UX 정의 (카운트다운, 재촬영 허용 여부)
- [ ] 프로젝트 초기 세팅 (폴더 구조, 의존성, lint, CI)

## 5. 미정 / 사용자 확인 필요 사항

- 타깃 플랫폼: iOS만? Android만? 둘 다?
- 개인 프로젝트인지, 팀/상용을 염두에 둔 것인지
- 디자인 레퍼런스가 있는지
- 데드라인 또는 마일스톤
