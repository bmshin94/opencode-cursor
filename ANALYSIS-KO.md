# open-cursor 전수조사 분석 리포트 🔍

> 작성: 2026-09-19 · 대화 정리본
> 분석 대상 저장소: <https://github.com/bmshin94/opencode-cursor>
> 원본(upstream): <https://github.com/Nomadcxx/opencode-cursor>
> npm 패키지: <https://www.npmjs.com/package/@rama_nigg/open-cursor>
> 공식 문서: <https://nomadcxx.github.io/opencode-cursor/docs/>

---

## 목차

1. [프로젝트 정체](#1-프로젝트-정체)
2. [쉬운 설명 (비유편)](#2-쉬운-설명-비유편)
3. [질의응답 7선](#3-질의응답-7선)
4. [수익화 아이디어](#4-수익화-아이디어)
5. [참고 링크 모음](#5-참고-링크-모음)

---

## 1. 프로젝트 정체

### 한 줄 요약

> **Cursor 구독으로 쓸 수 있는 모델들을, 터미널 코딩 에이전트 OpenCode 안에서 그대로 사용하게 해주는 어댑터 플러그인.**

- 패키지명: `@rama_nigg/open-cursor`
- 버전: `2.5.8`
- 작성자: Nomadcxx (본 저장소는 `bmshin94` 포크)
- 설명(원문): *"No prompt limits. No broken streams. Full thinking + tool support. Your Cursor subscription, properly integrated."*

### 왜 필요한가

| | Cursor | OpenCode |
|---|---|---|
| 형태 | GUI 에디터 (VSCode 포크) | 터미널 CLI 에이전트 |
| 모델 | 구독 하나로 다수 모델 사용 | 모델별 API 키를 따로 구매 |
| 문제 | 에디터 밖에선 사용 불가 | API 비용이 별도로 발생 |

이 프로젝트는 그 사이에 끼어들어 **"OpenAI 호환 API 서버인 척"** 하면서 실제 요청은 Cursor로 넘긴다.

### 저장소 구조 (전체 280개 파일)

```
opencode-cursor/
├── src/           핵심 로직 (TypeScript, 약 16,600줄)
├── cmd/installer/ Go + bubbletea 기반 터미널 설치 UI
├── docs-site/     Next.js + fumadocs 문서 사이트 (MDX 24개)
├── tests/         테스트 86개 파일 (unit / integration)
├── scripts/       sdk-runner.mjs, cursor-agent-runner.mjs 등
├── install.sh     curl 원라이너 설치 스크립트
└── .github/       CI, npm 배포, 문서 배포 워크플로
```

### `src/` 모듈별 역할

| 폴더 | 줄 수 | 역할 |
|---|---:|---|
| `provider/` | 2,954 | 경계선(boundary). Cursor 응답 → OpenAI 형식 번역, 툴 스키마 자동 수리, 반복 호출 차단 |
| `proxy/` | 1,663 | 로컬 HTTP 서버(`127.0.0.1:32124`), 프롬프트 빌더, 세션 재개 |
| `cli/` | 1,670 | `open-cursor install / status / doctor / sync-models` 명령 |
| `client/` | 1,391 | `cursor-agent` 프로세스 및 `@cursor/sdk` 자식 프로세스 관리 |
| `tools/` | 1,349 | 툴 레지스트리·라우터·실행기(local/cli/sdk/mcp) |
| `models/` | 1,084 | 모델 자동 발견, 가격표, 변형(variants) 관리 |
| `streaming/` | 563 | 라인 버퍼 → 파서 → SSE 변환 |
| `mcp/` | 397 | MCP 서버 연결 브리지 |
| `plugin.ts` | 3,102 | 단일 최대 파일. OpenCode 플러그인 진입점 |

### 요청 흐름

```
사용자 입력
   ↓
OpenCode (모델: cursor-acp/auto)
   ↓  OpenAI 호환 HTTP 요청
로컬 프록시 127.0.0.1:32124
   ↓
cursor-agent 프로세스 (기본)  또는  @cursor/sdk 러너 (대체)
   ↓
Cursor API → 실제 모델
   ↓  stream-json 이벤트
provider 경계선에서 번역
   ├─ 텍스트 / thinking → SSE 스트리밍
   └─ 툴 호출 → OpenCode 툴 호출
   ↓
터미널 출력
```

핵심: **OpenCode는 자신이 평범한 OpenAI 호환 프로바이더를 쓰고 있다고 인식한다.**

### 사용 시나리오

1. 터미널 중심 개발자 (vim/tmux 환경)
2. SSH 원격 서버 작업
3. API 비용 절감 (Cursor 구독 하나로 통합)
4. 자동화·스크립트 (`opencode run "..."` 을 CI/cron에 연결)
5. Cursor에 신규 모델 추가 시 자동 반영

### 학습 자료로서의 가치

- 프로토콜 어댑터 패턴 (A 시스템 ↔ B 시스템 통역)
- 스트리밍 파서 (NDJSON 조각 → SSE)
- 자식 프로세스 관리 및 프로세스 풀링
- LLM 툴 콜 루프 + 무한 루프 방어
- 32개 환경변수로 모든 경로를 토글하는 설계

### 주의사항 3가지

1. **`cursor-agent` 단종 예정** — CHANGELOG `[Unreleased]`에 따르면 Cursor IDE 0.43+ 에서 바이너리가 제거되었고, 공식 `@cursor/sdk` + API 키 방식으로 전환 중. README는 아직 구방식(`cursor-agent login`) 기준이라 불일치가 있다.
2. **라이선스 불일치** — README는 `BSD-3-Clause`, `package.json`은 `ISC`. 상업적 활용 전에 정리 필요.
3. **약관 회색지대** — Cursor 구독을 Cursor 외부에서 사용하는 구조라, 상업적 재판매는 리스크가 크다.

---

## 2. 쉬운 설명 (비유편)

### 넷플릭스 비유

프리미엄 구독(= Cursor)을 결제했는데 TV에서만 볼 수 있다. 차 안(= 터미널)에서도 보고 싶어서, **TV 신호를 차 화면용으로 바꿔주는 변환기**를 만든 것이 이 프로젝트다. 결제는 그대로, 보는 곳만 바뀐다.

### 통역사 비유 (가장 정확)

```
OpenCode   "OpenAI API 말" 만 가능
   ↕  open-cursor = 실시간 동시통역사
Cursor     "자기만의 프로세스 말" 만 가능
```

단순 번역을 넘어 **교정**까지 한다.

- Cursor가 파일 수정 명령을 잘못된 형식으로 보내면 → 올바른 스키마로 수리
- 같은 호출을 반복하면 → 루프로 판단해 차단
- 파일을 통째로 날릴 위험이 보이면 → 덮어쓰기 가드로 거부

### 카페 비유 (구조 이해용)

| 등장인물 | 실제 정체 |
|---|---|
| 손님 | 터미널 앞의 사용자 |
| 키오스크 | OpenCode |
| 주문 전달 창구 | 로컬 프록시 `:32124` |
| 주방 직원 | `cursor-agent` 프로세스 |
| 본사 주방 | Cursor 서버 (실제 모델) |
| 컨베이어 | 스트리밍 출력 |

### 어려운 문제를 푼 지점 3가지

**① 스트리밍 번역**
AI는 글자를 한 조각씩 보내는데 형식이 서로 다르다.
- Cursor: `{"type":"text","delta":"안"}` 줄바꿈 구분 (NDJSON)
- OpenAI: `data: {...}\n\n` (SSE)

게다가 네트워크로 오다가 `{"type":"te` 처럼 **반토막 난 상태**로 도착한다. `line-buffer.ts`가 조각을 모아 완전한 줄이 될 때까지 버퍼링한다.

**② 툴 호출의 소유권**
"이 파일 수정해"를 누가 실행할 것인가?
- Cursor가 실행 → OpenCode 권한 검사를 우회해 위험
- OpenCode가 실행 → 안전하지만 번역 필요

이 프로젝트는 **OpenCode 실행을 기본값**(`CURSOR_ACP_TOOL_LOOP_MODE=opencode`)으로 잡았다. 동시에 문서에서 *"Cursor 네이티브 툴이 먼저 워크스페이스를 변경하면 소급 적용할 수 없다"* 는 한계를 명시한다.

**③ 무한 루프 방지**
`tool-loop-guard.ts`(657줄)가 호출 지문(fingerprint)을 기록해 반복 호출을 감지하고 끊는다.

---

## 3. 질의응답 7선

### Q1. 설치 및 사용법

**사전 준비**

```bash
opencode --version
cursor-agent --version
```

**설치**

```bash
npm install -g @rama_nigg/open-cursor
open-cursor install
```

`install`이 자동으로 수행하는 것:
1. `opencode.json`에 `cursor-acp` 프로바이더 추가
2. OpenCode 플러그인 디렉터리에 플러그인 설치
3. Cursor가 노출하는 모델 목록 발견
4. OpenAI 호환 프로바이더 패키지 설치

기존 설정은 타임스탬프 백업 후 덮어쓴다(`--no-backup`으로 생략 가능).

**로그인 및 확인**

```bash
cursor-agent login
opencode models | grep cursor-acp   # cursor-acp/auto 가 나오면 성공
```

**사용**

```bash
opencode run "이 저장소를 5줄로 요약해줘" --model cursor-acp/auto
opencode   # 대화형 실행 후 모델 선택기에서 cursor-acp/* 선택
```

**주요 명령어**

| 명령 | 용도 |
|---|---|
| `open-cursor status` | 설치·인증·백엔드 상태 확인 |
| `open-cursor doctor` | 문제 진단 및 복구 명령 제시 |
| `open-cursor doctor --deep` | 정밀 검사 |
| `open-cursor sync-models` | 모델 목록 갱신 |
| `open-cursor models --explain` | 모델 그룹핑 설명 |
| `open-cursor uninstall` | 제거 |

**설정 파일 위치**
- Linux / macOS: `~/.config/opencode/opencode.json`
- Windows: `%USERPROFILE%\.config\opencode\opencode.json`

**대체 설치 경로**

```bash
# 셸 설치기 (Linux/macOS) — Go TUI 설치 UI
curl -fsSL https://raw.githubusercontent.com/Nomadcxx/opencode-cursor/main/install.sh | bash

# 소스 설치 (개발용)
git clone https://github.com/bmshin94/opencode-cursor.git
cd opencode-cursor
./scripts/install-plugin.sh
```

**디버깅**

```bash
CURSOR_ACP_LOG_LEVEL=debug CURSOR_ACP_LOG_CONSOLE=1 \
  opencode run "test" --model cursor-acp/auto
```

로그 기본 경로: `~/.opencode-cursor/`

---

### Q2. 플러그인인가, 스킬인가, MCP인가?

**정답: OpenCode 플러그인 + 모델 프로바이더 (npm 패키지 형태)**

| 구분 | 해당 여부 | 근거 |
|---|:---:|---|
| 플러그인 | O | `package.json`의 `main: dist/plugin-entry.js`, OpenCode 플러그인 디렉터리에 설치 |
| 프로바이더 | O | `opencode.json`에 `provider.cursor-acp` 로 등록 |
| 스킬 | X | Claude Skill은 마크다운 지침. 이건 실행되는 TS 코드 (다만 `src/tools/skills/`로 OpenCode 스킬을 *지원*함) |
| MCP 서버 | X | MCP **서버가 아니라 클라이언트**. 설정된 MCP 서버를 발견해 모델에게 노출 |

**MCP와의 관계**

```
[MCP 서버들]  ←  open-cursor (클라이언트로 연결)
 filesystem / github / playwright ...
        ↓  mcp__<서버>__<툴> 형태로 이름 변환
   Cursor 모델에게 사용 가능한 툴로 주입
```

- `src/mcp/client-manager.ts` — MCP 서버 연결
- `mcptool` CLI — 셸에서 MCP 툴을 호출하는 다리
- `CURSOR_ACP_MCP_BRIDGE=false` 로 비활성화 가능

**ACP는?**
`cursor-acp`의 ACP = Agent Client Protocol. 다만 로드맵상 **네이티브 ACP는 보류(Deferred)** 상태이고, 현재는 프로바이더 이름에만 남아 있다.

> 정리: **플러그인이자 프로바이더이며 동시에 MCP 클라이언트인 하이브리드.**

---

### Q3. API 토큰이 필요한가?

**현재(v2.5.8) 기준으로는 필수가 아니다. 다만 곧 필요해진다.**

| 방식 | 명령 | API 키 | 상태 |
|---|---|:---:|---|
| cursor-agent 로그인 | `cursor-agent login` | 불필요 | 기본값·권장 |
| OpenCode auth | `opencode auth login --provider cursor-acp` | 상황에 따라 | SDK 대비 |
| SDK 백엔드 | `CURSOR_API_KEY=...` | 필수 | 선택·대체 |

기본 `CURSOR_ACP_BACKEND=auto` 동작:

```
cursor-agent 존재 → 그것을 사용 (API 키 불필요)
        ↓ 없으면
실제 API 키 존재 → SDK 백엔드로 폴백
```

**예정된 변경 (CHANGELOG `[Unreleased]`, BREAKING)**

- Cursor가 IDE 0.43+ 에서 `cursor-agent` 바이너리 제거
- 공식 `@cursor/sdk`로 전환
- `cursor-agent login` OAuth 방식 **지원 종료**
- 키 우선순위: `CURSOR_API_KEY` → OpenCode auth store → `opencode.json`의 `provider.cursor-acp.options.apiKey`

**API 키 발급**

1. <https://cursor.com/settings> 접속 후 발급
2. 설정:

```bash
export CURSOR_API_KEY=<실제_키>
export CURSOR_ACP_BACKEND=sdk
```

> 주의: 구문서의 `cursor-agent` 라는 플레이스홀더 문자열을 SDK 키로 사용하면 안 된다.

**비용 구조**
플러그인 자체는 오픈소스라 무료. 비용은 **Cursor 구독 한도**에서 소모되며, 별도 LLM API 과금은 발생하지 않는다.

---

### Q4. 왜 GitHub에서 유명한가?

**① 돈 문제를 정확히 겨냥**
Claude API를 직접 헤비하게 쓰면 월 수백 달러, Cursor 구독은 월 $20 수준. "이미 낸 구독료를 최대한 활용한다"는 보편적 니즈를 직격했다.

**② 경쟁 프로젝트 대비 품질 우위**
문서에 경쟁 프로젝트 6개를 직접 비교하는 표가 있다.

| 프로젝트 | 접근법 | 주요 한계 |
|---|---|---|
| stablekernel/opencode-cursor | 공식 `@cursor/sdk` | 권한 밖 툴 실행 위험 경고 |
| cursor-opencode-provider | Connect-RPC / protobuf | 프로토콜 변경 시 깨짐 |
| yet-another-opencode-cursor-auth | 비공식 인터페이스 | 계정 리스크 명시 |
| opencode-cursor-auth | 로컬 서비스 | 툴 콜 실험적, thinking·usage 누락 |
| cursor-opencode-auth | 독립 프록시 | macOS 전용 |
| **open-cursor** | cursor-agent + SDK 폴백 | 공식 CLI 경로 유지 |

경쟁자를 숨기지 않고 정직하게 비교하는 점이 신뢰를 준다.

**③ 문서 품질**
- Next.js + fumadocs 기반 전용 문서 사이트
- MDX 문서 24개 (설치/인증/가이드/레퍼런스/아키텍처/개발)
- 32개 환경변수 전부 표로 정리
- `llms.txt`, `llms-full.txt` 제공 (AI 친화적)
- **문서 빌드 시 소스와 대조해 환경변수 누락이 있으면 CI 실패**

**④ 완성도 있는 디테일**
- Go + bubbletea 터미널 설치 UI (애니메이션 포함)
- 로고 SVG, 배너, OG 이미지 자동 생성
- Windows / macOS / Linux 전 플랫폼 지원
- CI에서 unit / integration 분리 실행 + Step Summary 출력

**⑤ 활발한 커뮤니티**
외부 기여 PR이 계속 머지되고 있으며(#126, #127, #128 등), 이슈 번호가 128번대라는 점은 실사용자가 많다는 신호다.

**⑥ 한계를 솔직히 밝힘**
*"Cursor 네이티브 툴이 OpenCode가 이벤트를 보기 전에 워크스페이스를 변경하면, 프로바이더 번역기는 OpenCode 권한을 소급 적용할 수 없다"* 는 문장을 문서에 명시한다.

---

### Q5. 로컬 에이전트 구축에 도움이 되는가?

**매우 도움이 된다. 사실상 교과서 수준이다.**

**재사용 가능한 패턴 7가지**

1. **어댑터 패턴** — 어떤 AI든 로컬 프록시를 거쳐 OpenAI 호환 엔드포인트로 노출. Ollama, vLLM, 사내 LLM에 그대로 적용 가능.
2. **스트리밍 파서** (`src/streaming/`, 563줄) — `line-buffer.ts`(반토막 JSON 재조립), `delta-tracker.ts`(중복 제거), `openai-sse.ts`(SSE 출력).
3. **툴 콜 루프 + 루프 가드** (`tool-loop-guard.ts`, 657줄).
4. **자식 프로세스 관리** (`client/`) — 프로세스 풀링, 유휴 자동 종료(기본 15분), Windows/Unix 셸 차이 처리.
5. **스키마 자동 수리** (`tool-schema-compat.ts`, 1,004줄) — 모델이 틀린 인자 형식을 보낼 때 자동 교정. 소형 모델 사용 시 특히 유용.
6. **MCP 브리지** (`src/mcp/`) — 서버 발견 → 툴 이름 네임스페이싱 → 모델에 주입.
7. **환경변수 기반 토글 설계** — 32개 변수로 모든 경로를 켜고 끌 수 있어 디버깅이 쉽다.

**조립 가능한 구조**

```
내 로컬 에이전트
 ├─ 모델층  ← 어댑터 패턴으로 Cursor / Ollama / Claude 전환
 ├─ 툴층    ← MCP 브리지 패턴
 ├─ 루프층  ← tool-loop-guard 패턴
 └─ UI층    ← 웹(React) 또는 CLI
```

**읽는 순서 권장**
`streaming/`(563줄) → `proxy/` → `provider/` → `plugin.ts`(3,102줄은 마지막).
전체 16,600줄을 한 번에 이해하려 하면 지친다. 또한 Bun 기반이라 Node 전용 환경이면 일부 수정이 필요하다.

---

### Q6. 수익화 아이디어가 있는가?

요약 목록 (상세는 4장 참조):

1. Universal LLM Gateway (SaaS)
2. 팀용 AI 비용 관리 대시보드
3. 셀프호스팅 에이전트 플랫폼
4. 교육 콘텐츠 / 강의
5. 도메인 특화 에이전트
6. 개발자 도구 Micro-SaaS

**금지선: "Cursor 구독 공유/재판매 서비스"** — 약관 위반, 계정 정지, 법적 리스크.

---

### Q7. React나 PHP로 만들 수 있는가?

**부분적으로 가능하다. 역할을 나누는 것이 정답이다.**

핵심 기능이 **장시간 상주하는 로컬 프로세스**(자식 프로세스 spawn, 로컬 포트 바인딩, 프로세스 풀 유지)라서 그대로 포팅하기는 어렵다.

| 기술 | 가능 여부 | 이유 |
|---|:---:|---|
| React (브라우저) | X | 프로세스 spawn·포트 바인딩 불가 |
| React Native / Electron | O | Electron 메인 프로세스에서 Node API 사용 가능 |
| PHP (웹서버 모드) | △ | 요청 단위로 종료되어 상주 프로세스 유지 곤란 |
| PHP CLI + ReactPHP / Swoole | O | 비동기 이벤트 루프 지원 |
| Node / Bun | O | 원 구현 |

**권장 아키텍처**

```
React 프론트엔드 (채팅 / 대시보드 / 설정, SSE 수신)
        ↓ HTTP · SSE
PHP(Laravel) 백엔드 (인증 / 유저 / 결제 / 로그)
        ↓ 내부 HTTP
Node·Bun 엔진 (이 프로젝트 재사용: 프로세스 / 스트림)
```

**React로 만들 수 있는 것**
- 환경변수 32개 GUI 설정 대시보드
- 채팅 UI — 프록시가 OpenAI 호환이므로 Vercel AI SDK 직결

```jsx
const { messages } = useChat({ api: 'http://127.0.0.1:32124/v1/chat/completions' })
```

- 사용량 시각화 (`src/usage.ts`, `src/models/pricing.ts` 활용)
- 로그 뷰어 (`~/.opencode-cursor/` 실시간 표시)

**PHP로 만들 수 있는 것**
- Laravel 기반 팀 게이트웨이 (인증 / 쿼터 / 과금)
- 사용량 집계 API 및 부서별 비용 리포트
- 웹훅 서버 (GitHub PR → 자동 리뷰 트리거)
- 대화 이력 저장소 (MySQL 세션 보관)

**결론:** 엔진은 Node/Bun을 그대로 쓰고, React·PHP는 사용자 접점에 집중하는 것이 가장 빠르다.

---

## 4. 수익화 아이디어

### 전제: 하지 말아야 할 것

> **"Cursor 구독 공유 / 재판매 서비스"는 금지.**
> - Cursor 약관 위반 (계정 공유 금지)
> - 대량 계정 정지 → 전체 유저 피해 → 환불 리스크
> - 법적 분쟁 가능성
> - Cursor의 API 변경 한 번으로 사업이 즉시 중단될 수 있음

---

### 아이디어 1. Universal LLM Gateway (SaaS)

이 프로젝트의 어댑터 패턴을 일반화해, 모든 LLM을 하나의 OpenAI 호환 엔드포인트로 통합.

```
Ollama / Claude / GPT / Gemini / 사내 LLM
        ↓
   내 게이트웨이  →  OpenAI 호환 API 하나
        ↓
 라우팅 · 캐싱 · 폴백 · 비용 추적 · 로깅
```

**차별화**
- 스마트 라우팅: 난이도에 따라 저가/고가 모델 자동 배분 → 비용 40~70% 절감
- 자동 폴백: 한 모델 장애 시 즉시 전환
- 프롬프트 캐싱
- 팀·프로젝트별 통합 비용 대시보드

**가격 모델**

| 플랜 | 가격 | 대상 |
|---|---|---|
| Free | $0 | 월 1만 요청 |
| Pro | $29/월 | 개인 개발자 |
| Team | $99/월 | 5~20인 팀 |
| Enterprise | 협의 | 온프레미스 + SLA |

**평가**
- 난이도 중상 / 시장 매우 큼 / 경쟁 치열 (LiteLLM, OpenRouter, Helicone)
- 니치 전략 필요: "국내 기업 특화 + 온프레미스 + 한국어 지원"

---

### 아이디어 2. AI 비용 관리 SaaS (FinOps for AI) — **1순위 추천**

핵심 문제: **AI를 쓰는 회사 대부분이 자기가 얼마를 쓰는지 모른다.**

**기능**
- 직원별 / 팀별 사용량 추적
- 예산 초과 알림 (Slack / 이메일)
- 낭비 탐지: "간단한 질문에 최고가 모델 사용 중"
- 최적화 제안: "이 작업을 저가 모델로 전환 시 월 X원 절감"
- 감사 로그: 누가 언제 어떤 데이터를 AI에 입력했는지 (컴플라이언스)

**이 저장소에서 가져올 자산**
- `src/usage.ts` — 토큰 사용량 추적
- `src/models/pricing.ts` — 모델별 가격표
- 프록시 구조 — 모든 요청이 통과하므로 자동 집계 가능

**가격**: 사용량의 3~5% 또는 시트당 $10~20/월
**세일즈 포인트**: "AI 비용 30% 절감, 절감액의 10%만 과금"

**평가**
- 난이도 중 / 시장 급성장 / **국내 경쟁 거의 없음**
- React + PHP 조합으로 충분히 구현 가능 (Q7과 직결)

---

### 아이디어 3. 셀프호스팅 코딩 에이전트 플랫폼

보안 규제로 클라우드 AI를 쓸 수 없는 조직(금융·의료·공공·방산) 타깃.

```
사내 서버
 ├─ 이 프로젝트 구조 기반 에이전트 엔진
 ├─ 온프레미스 LLM (Llama, Qwen 등)
 ├─ 사내 코드베이스 RAG
 └─ React 웹 UI + 권한 관리
```

**가격**: 라이선스 연 2,000만~1억원 + 구축 컨설팅 별도 + 유지보수(라이선스의 20%)
**평가**: 난이도 높음 / 건당 금액 큼 / 기술력보다 B2B 영업력이 관건

---

### 아이디어 4. 교육 & 콘텐츠 — **2순위 추천 (즉시 시작 가능)**

| 상품 | 가격대 | 비고 |
|---|---|---|
| 전자책 「AI 에이전트 직접 만들기」 | 3~5만원 | 이 코드 분석 기반 |
| 온라인 강의 (인프런 / 유데미) | 10~20만원 | 어댑터·스트리밍·툴 루프 |
| 기업 출강 | 일 100~300만원 | 사내 AI 에이전트 구축 실무 |
| 유료 뉴스레터 | 월 1만원 | AI 툴 생태계 주간 리포트 |
| 부트캠프 | 300~500만원 | 4주 과정 |

**시작 전략**
1. 블로그·유튜브 무료 콘텐츠 (예: "Cursor 구독으로 터미널 AI 쓰는 법")
2. 반응 확인 후 전자책
3. 전자책 판매 실적 기반으로 강의
4. 강의 실적 기반으로 기업 출강

**평가**: 난이도 낮음 / 리스크 거의 없음 / 가장 빠른 시작점

---

### 아이디어 5. 도메인 특화 에이전트

| 에이전트 | 기능 | 가격대 |
|---|---|---|
| 레거시 코드 분석기 | 구형 PHP/JSP 문서화 + 마이그레이션 계획 | 프로젝트당 500만~2,000만원 |
| 보안 감사 봇 | PR 단위 취약점 자동 검사 | 리포당 $199/월 |
| 테스트 생성기 | 커버리지 취약 지점 자동 테스트 작성 | $99/월 |
| 문서 자동화 | 코드 → API 문서 생성·갱신 | $49/월 |
| i18n 에이전트 | 다국어 리소스 자동 관리 | $79/월 |

**평가**: 시장은 좁지만 지불 의사가 높다. 본인 전문 도메인이 있으면 강력 추천.

---

### 아이디어 6. 개발자 도구 (Micro-SaaS)

- AI 설정 관리 GUI ($9/월)
- AI 요청 디버거 — 프롬프트/응답/토큰 시각화 ($19/월)
- 프롬프트 A/B 테스터 ($29/월)
- **MCP 서버 마켓플레이스** — 검색/설치/관리 허브 (수수료 모델). MCP 생태계 확장기라 선점 가치가 있다.

---

### 실행 로드맵

**Phase 1 (1~2개월) — 인지도, 비용 0원**
- 이 프로젝트 심층 분석 블로그 시리즈 5편
- "Cursor 구독으로 터미널 AI 쓰기" 실용 글
- 원본 저장소에 PR 1~2건 기여
- SNS로 개발 과정 공유

**Phase 2 (3~4개월) — 첫 수익**
- 전자책 출간
- 무료 툴 출시 → 이메일 리스트 수집
- 유료 뉴스레터 시작

**Phase 3 (5~12개월) — 제품화**
- 아이디어 2번 MVP 개발
- 베타 10팀 무료 제공 → 피드백
- 유료 전환, 첫 MRR 확보

**Phase 4 (1년+) — 확장**
- 기업 고객 확보
- 팀 빌딩 또는 투자 유치

---

### 최종 추천

**1순위: AI 비용 관리 SaaS (아이디어 2)**
- 이 프로젝트 구조를 가장 직접적으로 재활용
- 국내 시장 공백
- ROI 증명이 쉬워 영업이 용이
- React + PHP로 구현 가능

**2순위: 교육 콘텐츠 (아이디어 4)**
- 즉시 시작 가능, 리스크 없음
- 1순위 제품의 마케팅 채널 역할

> 두 가지를 병행하는 것이 최적. 교육 콘텐츠로 신뢰를 쌓고, 그 독자를 SaaS 초기 고객으로 전환한다.

---

## 5. 참고 링크 모음

### 저장소 · 패키지

| 항목 | 링크 |
|---|---|
| **본 저장소 (포크)** | <https://github.com/bmshin94/opencode-cursor> |
| **원본 저장소 (upstream)** | <https://github.com/Nomadcxx/opencode-cursor> |
| npm 패키지 | <https://www.npmjs.com/package/@rama_nigg/open-cursor> |
| 이슈 트래커 | <https://github.com/Nomadcxx/opencode-cursor/issues> |
| 설치 스크립트 | <https://raw.githubusercontent.com/Nomadcxx/opencode-cursor/main/install.sh> |
| 작성자 GitHub | <https://github.com/Nomadcxx> |
| 스폰서 | <https://github.com/sponsors/Nomadcxx> |

### 공식 문서

| 문서 | 링크 |
|---|---|
| 문서 홈 | <https://nomadcxx.github.io/opencode-cursor/docs/> |
| 설치 | <https://nomadcxx.github.io/opencode-cursor/docs/getting-started/installation/> |
| 인증 | <https://nomadcxx.github.io/opencode-cursor/docs/getting-started/authentication/> |
| 빠른 시작 | <https://nomadcxx.github.io/opencode-cursor/docs/getting-started/quick-start/> |
| 트러블슈팅 | <https://nomadcxx.github.io/opencode-cursor/docs/getting-started/troubleshooting/> |
| 설정 레퍼런스 | <https://nomadcxx.github.io/opencode-cursor/docs/reference/configuration/> |
| CLI 레퍼런스 | <https://nomadcxx.github.io/opencode-cursor/docs/reference/cli/> |
| 대안 비교 | <https://nomadcxx.github.io/opencode-cursor/docs/reference/alternatives/> |
| 로드맵 | <https://nomadcxx.github.io/opencode-cursor/docs/reference/roadmap/> |
| 아키텍처 개요 | <https://nomadcxx.github.io/opencode-cursor/docs/architecture/overview/> |
| 스트림 번역 | <https://nomadcxx.github.io/opencode-cursor/docs/architecture/stream-translation/> |
| 툴 루프 | <https://nomadcxx.github.io/opencode-cursor/docs/architecture/tool-loop/> |
| MCP 가이드 | <https://nomadcxx.github.io/opencode-cursor/docs/guides/mcp-servers/> |

### 경쟁 · 대안 프로젝트

| 프로젝트 | 링크 |
|---|---|
| stablekernel/opencode-cursor | <https://github.com/stablekernel/opencode-cursor> |
| cursor-opencode-provider | <https://github.com/oakimov/cursor-opencode-provider> |
| yet-another-opencode-cursor-auth | <https://github.com/Yukaii/yet-another-opencode-cursor-auth> |
| opencode-cursor-auth | <https://github.com/POSO-PocketSolutions/opencode-cursor-auth> |
| cursor-opencode-auth | <https://github.com/R44VC0RP/cursor-opencode-auth> |

### 기타

- Cursor 설정 / API 키 발급: <https://cursor.com/settings>

---

*이 문서는 저장소 전체(280개 파일)를 직접 조사하여 작성한 분석 정리본입니다.*
