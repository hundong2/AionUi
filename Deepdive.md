# AionUi 코드 Deep Dive (입문자용)

이 문서는 **처음 오픈소스 컨트리뷰션을 시작하는 사람**이 AionUi 코드를 빠르게 이해하고,
실제로 버그를 찾고 PR을 올릴 수 있도록 돕기 위한 실전 가이드입니다.

---

## 1) 먼저 이해해야 할 목표

입문자의 1차 목표는 다음 3가지입니다.

1. **실행 경로를 이해한다**: 앱이 어디서 시작해서 어디로 흐르는지
2. **재현 가능한 버그를 찾는다**: “증상 → 원인 후보 → 검증” 루프를 만든다
3. **작은 PR 1개를 완성한다**: 작고 안전한 수정 + 테스트/검증 + 설명

즉, “대규모 리팩토링”이 아니라, **작은 신뢰 가능한 수정**이 첫 기여 목표입니다.

---

## 2) 프로젝트를 큰 그림으로 보기

AionUi는 Electron 기반 멀티 프로세스 앱입니다.

- **Main Process**: 앱 생명주기/창 생성/브리지 초기화
- **Preload**: Renderer와 Main 간 보안 IPC 브리지
- **Renderer(React)**: 실제 UI
- **Process layer**: DB, 브리지 핸들러, 에이전트 매니저, 서비스
- **WebUI server(선택)**: 원격 접속용 Express + WebSocket

핵심 진입 파일:

- `src/index.ts`
- `src/preload.ts`
- `src/renderer/main.tsx`
- `src/process/index.ts`
- `src/process/bridge/index.ts`

---

## 3) 실행 흐름 (처음 읽을 때 이 순서 추천)

### Step A. Electron 시작점

`src/index.ts`

- `createWindow()`에서 BrowserWindow 생성
- `preload` 연결
- 앱 준비 후 `initializeProcess()` 호출
- 웹 UI 모드(`--webui`), 원격 모드(`--remote`) 같은 실행 옵션 처리

### Step B. Renderer와 통신 방법

`src/preload.ts`

- `contextBridge.exposeInMainWorld('electronAPI', ...)`로 안전한 API 노출
- `emit/on` 기반 IPC 호출
- WebUI 비밀번호 변경/QR 토큰 생성 같은 direct IPC도 존재

### Step C. 실제 비즈니스 초기화

`src/process/index.ts`, `src/process/initBridge.ts`

- DB 초기화(`initStorage`)
- 브리지 초기화(`initAllBridges`)
- 크론 서비스 초기화
- 채널 매니저 초기화

### Step D. 어떤 브리지가 있는지 확인

`src/process/bridge/index.ts`

- conversation, model, mcp, auth, fs, cron, webui 등 브리지 등록
- “문제가 어디 레이어인지” 추적할 때 출발점으로 매우 중요

---

## 4) 버그 탐색 기본 전략 (입문자용 템플릿)

버그를 찾을 때는 아래 템플릿을 고정으로 쓰면 좋습니다.

1. **증상 정의**: 무엇이 기대와 다른가?
2. **재현 절차 작성**: 클릭/입력/환경/명령어를 숫자로 적기
3. **로그 관찰 지점 지정**: main 로그인지, renderer 콘솔인지, 브리지 예외인지
4. **원인 후보 2~3개 설정**
5. **작은 실험으로 후보 제거**
6. **수정은 최소 라인으로**
7. **수정 전/후 검증 증거 남기기**

---

## 5) 시작하기 좋은 버그 헌팅 포인트 5곳

아래는 입문자가 접근하기 비교적 좋은 “실제 코드 포인트”입니다.

### 1) 메시지 스트리밍/중복/누락

- `src/process/database/StreamingMessageBuffer.ts`
- `src/renderer/messages/hooks.ts`

관찰 포인트:

- 스트리밍 중 메시지 순서가 틀어지는지
- 동일 토큰이 중복 append 되는지

### 2) 에이전트 감지/모드 전환 오류

- `src/renderer/hooks/useMultiAgentDetection.tsx`
- `src/process/bridge/conversationBridge.ts`

관찰 포인트:

- 설치된 CLI 에이전트 감지 결과와 UI 표시가 다른지
- 세션 전환 시 이전 상태가 누수되는지

### 3) 파일 드래그&드롭/워크스페이스 경로 처리

- `src/renderer/pages/conversation/workspace/hooks/useWorkspaceDragImport.ts`
- `src/process/bridge/fsBridge.ts`
- `src/preload.ts`

관찰 포인트:

- OS별 경로 처리(공백/한글/특수문자)
- drag import 후 실제 파일 반영 실패 여부

### 4) 스케줄 작업(Cron) 재시작 후 동작

- `src/process/services/cron/CronService.ts`
- `src/process/database/schema.ts`
- `src/index.ts` (resume 관련)

관찰 포인트:

- 앱 재시작 후 잡 복구 여부
- sleep/wake 뒤 스케줄 누락 여부

### 5) WebUI 인증/비밀번호 리셋 경로

- `src/process/bridge/webuiBridge.ts`
- `src/webserver/auth/service/AuthService.ts`
- `src/preload.ts`

관찰 포인트:

- 비밀번호 변경 이벤트가 실제 서버 상태와 동기화되는지
- QR 로그인 토큰 생성/만료 처리

---

## 6) 테스트/검증 루틴 (실전)

### 테스트 위치

- `tests/unit`
- 설정: `jest.config.js`

### 기본 명령어

```bash
cd AionUi
npm run lint
npm test -- --runInBand
```

참고:

- 현재 저장소는 lint 경고가 많은 편이라, **내가 건드린 범위에서 새 경고/에러를 만들지 않는 것**이 중요합니다.
- 첫 PR은 테스트 추가가 어렵다면, 최소한 재현 절차와 수동 검증 결과를 PR 본문에 명확히 남기세요.

---

## 7) 입문자용 첫 컨트리뷰션 플랜 (7일 예시)

### Day 1: 빌드/실행/구조 파악

- 앱 실행
- `src/index.ts → preload.ts → process/bridge/index.ts` 흐름 읽기

### Day 2: 버그 1개 선택 + 재현 시나리오 작성

- 재현 단계를 숫자로 고정
- “기대 결과 vs 실제 결과”를 1문장으로 정의

### Day 3: 원인 후보 축소

- 관련 파일 2~3개만 추려서 추적
- 콘솔/IPC 에러 로그 확인

### Day 4: 최소 수정

- 함수 1~2개, 라인 최소 변경
- 부수 효과 없는 방향 우선

### Day 5: 검증

- 수정 전/후 비교
- 대상 테스트(가능하면 unit) 실행

### Day 6: PR 작성

- 재현 절차
- 수정 이유
- 검증 방법
- 리스크/미해결 항목

### Day 7: 리뷰 반영

- 지적받은 부분만 좁게 수정
- 새 범위 확장 금지

---

## 8) 코드 읽을 때 자주 헷갈리는 지점

1. **Bridge 파일이 많다**
   - 해결: `src/process/bridge/index.ts`에서 등록 순서를 먼저 보고 개별 브리지로 내려가기

2. **Renderer에서 직접 비즈니스 처리한다고 착각**
   - 실제 핵심 로직은 process/main 쪽에 많은 경우가 많음

3. **"버그처럼 보이지만 정책"인 경우**
   - 예: fallback, silent catch, startup 시 graceful failure 설계

---

## 9) 좋은 첫 PR의 조건

- 변경 파일 수가 적다 (가능하면 1~3개)
- 재현 절차가 명확하다
- 수정 이유가 코드 레벨에서 설명된다
- 검증이 재실행 가능하다
- 부작용 범위가 작다

---

## 10) 바로 실행 가능한 체크리스트

- [ ] 내 로컬에서 버그를 100% 재현했다
- [ ] 원인 후보를 1개로 좁혔다
- [ ] 최소 변경으로 수정했다
- [ ] lint/test(또는 대상 테스트)를 실행했다
- [ ] PR 본문에 재현/수정/검증을 모두 적었다

이 체크리스트를 충족하면, 입문자여도 충분히 의미 있는 컨트리뷰션을 만들 수 있습니다.
