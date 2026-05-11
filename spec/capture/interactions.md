# capture — Interactions

## 진입
- params: `groupId`, (옵션) `slot: ISO timestamp` — 미전달 시 현재 슬롯 사용
- 진입 즉시:
  1. 카메라/마이크 권한 체크
  2. 현재 시각 vs 슬롯 마감 시각 비교 (D-12-02 grace 30초 적용)
  3. 권한 OK + 시간 OK → 대기 상태로 진입
  4. 권한 없음 → 권한 안내 화면
  5. 슬롯 마감 → 마감 안내 화면

## 슬롯 시간 검증
- 현재 슬롯 시작: `slotStart = toHourSlot(new Date())`
- 슬롯 마감: `slotEnd = slotStart + 1h + 30s` (grace 포함)
- `Date.now() > slotEnd` → 슬롯 마감 화면

## 카운트다운 흐름
1. 셔터 탭 → 햅틱 (medium impact) + 카운트다운 시작
2. 1초 간격 `3` → `2` → `1` 표시
3. 각 숫자 표시 시 햅틱 (light)
4. 카운트다운 종료 → 자동으로 녹화 시작

## 녹화 흐름
1. `cameraRef.recordAsync({ maxDuration: 3, codec: 'avc1', quality: '720p' })` 호출
2. 3초 진행 바 시각화
3. 3초 도달 시 자동 종료 → URI 획득
4. 미리보기 화면 전환

## 백그라운드 전환 / 인터럽트 처리

### 카운트다운 중
- `AppState`가 `inactive` 또는 `background`로 변경 시 카운트다운 즉시 취소
- 포그라운드 복귀 시 대기 상태로 리셋

### 녹화 중
- 백그라운드 전환 시 녹화 중단 + URI 폐기
- 전화 수신 시 (iOS 카메라 세션 자동 중단) → 녹화 중단
- 포그라운드 복귀 시 미리보기 진입하지 않고 대기 상태로 복귀

### 미리보기 중
- 백그라운드 전환 시 미리보기 상태 유지 (URI는 임시 파일이므로 OS가 정리 가능)
- 포그라운드 복귀 시 URI 유효성 재확인 → 무효면 대기 상태로

## "다시 찍기" 탭
- 현재 미리보기 URI 삭제
- 카운트다운 → 녹화 다시 시작
- 재촬영 횟수 제한 없음 (D-12-07: 슬롯 마감 전 무제한)

## "올리기" 탭
1. CTA 비활성 + 스피너 표시
2. 업로드 큐에 항목 등록 (D-12-07):
   ```
   {
     id: uuid(),
     groupId,
     slotKey: toHourSlot(slotStart),
     uid,
     localUri: uri,
     status: 'pending',
     retries: 0,
     createdAt: new Date(),
   }
   ```
3. AsyncStorage에 큐 저장 (영속화)
4. expo-task-manager 백그라운드 업로드 트리거
5. 즉시 그룹 상세로 복귀 (업로드는 백그라운드에서 진행)
6. 그룹 상세 화면에서 본인 슬롯에 "업로드 중..." placeholder 표시

## 슬롯 마감 직전 경고
- 남은 시간 5분 미만 진입 시:
  - 상단 카운트다운 빨간색
  - 햅틱 (warning impact)
- 남은 시간 30초 미만:
  - 셔터 영역 펄스 애니메이션
- 사용자가 미리보기 단계에서 슬롯 마감 도달 시:
  - 자동으로 "올리기" 트리거 (grace 30초 내에 업로드 시도)
  - 또는 사용자 경고 후 자동 폐기

## 슬롯 마감 후 진입 (외부에서)
- 그룹 상세 → "지금 촬영" 비활성 표시
- 캡처 화면 직접 진입 시도 시 마감 안내 후 자동 복귀

## 카메라 권한 거부 처리
- 첫 진입 시 권한 요청 → 거부 → 권한 안내 화면 표시
- "설정 열기" 탭 → `Linking.openSettings()` → 시스템 설정 → 권한 변경 후 앱 복귀
- `AppState` 변경 감지 → 권한 재체크 → OK면 자동으로 대기 상태로 전환

## 마이크 권한 거부 (Android)
- 마이크 권한만 거부된 경우: "소리 없이 촬영하시겠어요?" 다이얼로그
  - "소리 없이 촬영" → 진행
  - "권한 허용" → 설정 열기
- 사용자 선택에 따라 진행

## 저장공간 부족
- 녹화 시작 전 디바이스 free space < 10MB → 경고: "저장공간이 부족해요"
- 녹화 중 오류 발생 → 토스트 + 대기 상태 복귀

## 에러 처리
- 카메라 초기화 실패 → "카메라를 사용할 수 없어요" + 재시도 버튼
- 녹화 실패 → 토스트 "녹화에 실패했어요. 다시 시도해 주세요" + 대기 상태
- HEVC 강제 변환 실패 (D-12-11) → fallback으로 디바이스 기본 코덱 사용 + Sentry 로그

## 접근성
- 카운트다운 a11y assertive announce
- 녹화 시작/종료 햅틱 + a11y
- 슬롯 마감 도달 a11y: "이 시간 슬롯이 마감되었습니다. 그룹으로 돌아갑니다"

## 트래킹
- `capture_view` (entry source: 그룹 상세 / 푸시)
- `capture_permission_denied` (camera / microphone)
- `capture_shutter_tap`
- `capture_recorded` (성공/실패)
- `capture_retake` (재촬영 횟수)
- `capture_upload_queued` (큐 등록 성공)
- `capture_slot_expired` (마감 직전 도달)
