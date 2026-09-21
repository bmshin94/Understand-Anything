# Understand Anything — 전수조사 분석 & 수익화 리포트 (한국어)

> 작성일: 2026-09-21
> 대상 저장소: **https://github.com/bmshin94/Understand-Anything**
> 원본(업스트림): **https://github.com/Egonex-AI/Understand-Anything**
> 원작자: [Lum1104](https://github.com/Lum1104) / 현재 관리: [Egonex](https://github.com/Egonex-AI)
> 라이선스: MIT © Yuxiang Lin and Infinite Universe, Inc.
> 공식 홈페이지: https://understand-anything.com · 라이브 데모: https://understand-anything.com/demo/

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉽게 이해하기](#2-쉽게-이해하기)
3. [설치 및 사용법](#3-설치-및-사용법)
4. [플러그인 vs 스킬 vs MCP](#4-플러그인-vs-스킬-vs-mcp)
5. [API 토큰이 필요한가](#5-api-토큰이-필요한가)
6. [왜 GitHub에서 유명한가](#6-왜-github에서-유명한가)
7. [로컬 에이전트 구축에 주는 도움](#7-로컬-에이전트-구축에-주는-도움)
8. [React / PHP로 만들 수 있는가](#8-react--php로-만들-수-있는가)
9. [수익화 아이디어 7선](#9-수익화-아이디어-7선)
10. [참고 링크](#10-참고-링크)

---

## 1. 프로젝트 개요

### 한 줄 정의

**코드베이스를 통째로 분석해 "지식 그래프(Knowledge Graph)"를 만들고, 이를 인터랙티브 웹 대시보드로 시각화해주는 AI 코드 이해 도구.**

### 저장소 규모 (실측)

| 항목 | 수치 |
|---|---|
| 소스 코드 | 약 59,222줄 (ts/tsx/mjs/py) |
| 커밋 수 | 461 |
| 기여자 | 62명 |
| 테스트 파일 | 66개 |
| 스킬(슬래시 커맨드) | 9개 |
| 에이전트 정의 | 10개 (프롬프트 총 4,092줄) |
| 지원 언어 | 43종 |
| 지원 프레임워크 | 11종 |
| 지원 플랫폼 | 17종 |
| 플러그인 버전 | 2.9.7 |

### 폴더 구조

```
Understand-Anything/
├── .claude-plugin/          Claude Code 플러그인 매니페스트 (marketplace.json + plugin.json)
├── .cursor-plugin/          Cursor 자동 발견용 매니페스트
├── .copilot-plugin/         VS Code + GitHub Copilot 자동 발견용 매니페스트
├── install.sh / install.ps1 14개 플랫폼용 설치 스크립트
├── homepage/                Astro 기반 공식 홈페이지 소스
├── docs/                    설계 스펙, 벤치마크, 증분 분석 문서
├── tests/                   스킬 / 훅 / 설치 스크립트 / 벤치마크 테스트
├── scripts/                 대용량 그래프 생성기, 벤치마크 러너
└── understand-anything-plugin/          ← 실제 구현 전부
    ├── skills/              9개 슬래시 커맨드 정의 (SKILL.md)
    ├── agents/              10개 서브에이전트 정의 (.md)
    ├── hooks/               SessionStart / PostToolUse 자동 갱신 훅
    └── packages/
        ├── core/            분석 엔진 (tree-sitter, 스키마, 검색, 지문, 도메인)
        ├── dashboard/       React 19 + React Flow 대시보드 (컴포넌트 32개)
        ├── viewer/          Claude 없이 그래프만 보는 독립 실행 뷰어
        ├── tree-sitter-swift-wasm/
        └── tree-sitter-dart-wasm/
```

### 핵심 설계: 정적분석 + LLM 하이브리드

| 역할 | 담당 | 산출물 |
|---|---|---|
| **구조(사실)** | tree-sitter (WASM 파서) | import/export, 함수·클래스 정의, 호출부, 상속 관계. **같은 코드 → 항상 같은 결과** |
| **의미(해석)** | LLM (Claude 등) | 평문 요약, 태그, 아키텍처 레이어 분류, 비즈니스 도메인 매핑, 학습 투어 |

> 이 분리 덕분에 **구조 정보는 재현 가능하고 LLM 환각에 오염되지 않으며**, 의미 정보만 AI가 담당한다.
> 이 프로젝트의 가장 중요한 기술적 차별점.

### 멀티 에이전트 파이프라인

```
Phase 0  사전점검 — 증분/전체 판단, 플러그인 빌드 확인, worktree 리다이렉트
   ↓
[1] project-scanner        파일 스캔 + 언어/프레임워크 감지
[2] file-analyzer  ×5 병렬  함수·클래스·import 추출 → 노드/엣지 생성 (배치당 20~30파일)
[3] architecture-analyzer  아키텍처 레이어 식별 (API / Service / Data / UI / Utility)
[4] tour-builder           의존성 순서 기반 가이드 투어 생성
[5] graph-reviewer         그래프 완결성 / 참조 무결성 검증
   ↓
.ua/knowledge-graph.json  →  /understand-dashboard 자동 실행
```

추가 에이전트
- `domain-analyzer` — 비즈니스 도메인/플로우/스텝 추출 (`/understand-domain`)
- `article-analyzer` — 위키 문서의 엔티티·주장·암묵적 관계 추출 (`/understand-knowledge`)
- `design-analyzer` — Figma 디자인 시스템 분석 (`/understand-figma`)
- `assemble-reviewer`, `knowledge-graph-guide` — 조립/가이드 보조

### 데이터 모델 (`packages/core/src/types.ts`)

- **노드 타입 27종**
  - 코드 5: `file` `function` `class` `module` `concept`
  - 비코드 8: `config` `document` `service` `table` `endpoint` `pipeline` `schema` `resource`
  - 도메인 3: `domain` `flow` `step`
  - 지식 5: `article` `entity` `topic` `claim` `source`
  - 디자인 6: `page` `screen` `component` `componentSet` `instance` `token`
- **엣지 타입 38종 / 9 카테고리**
  - 구조: `imports` `exports` `contains` `inherits` `implements`
  - 행위: `calls` `subscribes` `publishes` `middleware`
  - 데이터 흐름: `reads_from` `writes_to` `transforms` `validates`
  - 의존: `depends_on` `tested_by` `configures`
  - 의미: `related` `similar_to`
  - 인프라: `deploys` `serves` `provisions` `triggers`
  - 스키마: `migrates` `documents` `routes` `defines_schema`
  - 도메인: `contains_flow` `flow_step` `cross_domain`
  - 지식: `cites` `contradicts` `builds_on` `exemplifies` `categorized_under` `authored_by`
  - 디자인: `instance_of` `variant_of` `uses_token`

### 제공 명령어 9종

| 명령어 | 용도 |
|---|---|
| `/understand` | 코드베이스 분석 → 지식 그래프 생성 (메인) |
| `/understand-dashboard` | 인터랙티브 대시보드 실행 |
| `/understand-chat` | 그래프 기반 코드베이스 Q&A |
| `/understand-diff` | 변경사항 영향 범위(임팩트) 분석 |
| `/understand-explain` | 특정 파일/함수 딥다이브 설명 |
| `/understand-onboard` | 신입 온보딩 가이드 자동 생성 |
| `/understand-domain` | 비즈니스 도메인/플로우/스텝 추출 |
| `/understand-knowledge` | Karpathy 패턴 위키 지식베이스 분석 |
| `/understand-figma` | Figma 파일 → 디자인 지식 그래프 |

### 언제 쓰는가

1. 새 팀/새 프로젝트에 투입돼 대규모 코드베이스를 빠르게 파악해야 할 때
2. PR 리뷰 전 변경의 파급 효과를 확인할 때 (`/understand-diff`)
3. 팀 온보딩 문서를 대체할 때 (그래프를 커밋 → 팀원은 무료 열람)
4. 문서 없는 레거시 코드의 아키텍처를 역공학할 때
5. 리팩토링 계획 수립 시 순환 의존성·구조 문제를 시각적으로 찾을 때

---

## 2. 쉽게 이해하기

### 비유 1 — 낯선 도시의 지도

- 기존 코드 읽기 = 길거리 건물을 하나씩 들어가 보는 것 (전체 모습을 영영 모름)
- Understand Anything = 드론으로 도시 전체 지도를 그리고, 각 건물 용도까지 설명해주는 것

### 비유 2 — 요리책 만들기

| 단계 | 에이전트 | 하는 일 |
|---|---|---|
| 장보기 | `project-scanner` | 냉장고에 뭐가 있는지 전부 확인 |
| 손질 | `file-analyzer` ×5 | 재료별로 다듬어 정리 (동시 작업) |
| 분류 | `architecture-analyzer` | 전채/메인/디저트 구분 |
| 순서 | `tour-builder` | 무엇을 먼저 해야 하는지 가이드 |
| 검수 | `graph-reviewer` | 빠진 재료가 없는지 최종 확인 |

### 왜 AI에게만 맡기지 않는가

- "A 파일이 B를 import한다" → tree-sitter가 문법 트리를 직접 파싱하므로 **100% 정확**
- "이 파일은 결제 검증 담당" → LLM이 읽고 해석

**뼈대는 기계, 살은 AI.** 그래서 결과를 신뢰할 수 있다.

### 결과물

단 하나의 파일 `.ua/knowledge-graph.json`.

```json
{
  "version": "...",
  "project": { "name": "내프로젝트", "languages": ["TypeScript"], "frameworks": ["React"] },
  "nodes": [
    { "id": "file:src/auth.ts", "type": "file",
      "summary": "JWT 토큰 발급과 검증을 담당", "complexity": "moderate", "tags": ["auth"] }
  ],
  "edges": [
    { "source": "file:src/api.ts", "target": "file:src/auth.ts",
      "type": "imports", "direction": "forward", "weight": 0.9 }
  ],
  "layers": [ ... ],
  "tour":   [ ... ]
}
```

React 대시보드가 이 JSON을 읽어 노드-엣지 그래프로 렌더링한다.

### 토큰(비용) 구조

- **첫 실행** — 전체 코드를 LLM이 읽으므로 토큰 소모가 큼
- **이후 실행** — 파일 지문(fingerprint) 해시로 변경 감지 → **바뀐 파일만** 재분석 (증분)
- **팀원** — 커밋된 그래프를 뷰어로 열람 → **LLM 호출 0회, 비용 0원**

---

## 3. 설치 및 사용법

### 요구사항

- Node.js >= 22 (독립 뷰어는 >= 18)
- pnpm >= 10

### Claude Code (네이티브)

```bash
/plugin marketplace add Egonex-AI/Understand-Anything
/plugin install understand-anything
```

### 기타 플랫폼 (원라인 설치)

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/Egonex-AI/Understand-Anything/main/install.sh | bash
curl -fsSL https://raw.githubusercontent.com/Egonex-AI/Understand-Anything/main/install.sh | bash -s codex

# Windows PowerShell
iwr -useb https://raw.githubusercontent.com/Egonex-AI/Understand-Anything/main/install.ps1 | iex
```

지원 플랫폼 값: `gemini` `codex` `opencode` `pi` `openclaw` `antigravity` `vibe` `vscode` `hermes` `cline` `kimi` `trae` `nanobot` `kiro`

업데이트 / 삭제
```bash
./install.sh --update
./install.sh --uninstall <platform>
```

> **주의:** Codex는 `/` 대신 `$` 접두사를 쓴다 (`$understand`).
> 접두사가 안 먹히는 플랫폼에서는 자연어로 "understand 스킬로 이 프로젝트를 분석해줘"라고 요청하면 된다.

### Cursor / VS Code + Copilot

레포를 클론해 열기만 하면 `.cursor-plugin/plugin.json`, `.copilot-plugin/plugin.json`으로 **자동 발견**된다.

### Copilot CLI

```bash
copilot plugin install Egonex-AI/Understand-Anything:understand-anything-plugin
```

### 실전 사용 흐름

```bash
/understand                              # 분석 (첫 실행)
/understand-dashboard                    # 대시보드 (보통 자동 실행)

/understand-chat 결제 흐름 어떻게 돼?
/understand-diff
/understand-explain src/auth/login.ts
/understand-onboard
/understand-domain

/understand --language ko                # 한국어로 생성
/understand --full                       # 전체 재분석
/understand --auto-update                # 커밋마다 자동 갱신 훅 설치
/understand --review                      # LLM 그래프 리뷰어 전체 실행
/understand src/frontend                 # 모노레포 부분 분석
/understand --exclude "tests/*,docs/*"   # 경로 제외
```

지원 언어 옵션: `en`(기본) `zh` `zh-TW` `ja` `ko` `ru` — 첫 실행 시 대화 언어를 감지해 확인을 묻고, 선택은 `.ua/config.json`에 저장된다.

### 팀 공유 (LLM 없이 열람)

```bash
# 1) 그래프 커밋 (intermediate/ 와 diff-overlay.json 은 제외)
echo ".ua/intermediate/"      >> .gitignore
echo ".ua/diff-overlay.json"  >> .gitignore
git add .ua/ && git commit -m "docs: add knowledge graph"

# 2) 팀원은 Node.js만 있으면 끝
npx https://github.com/Egonex-AI/Understand-Anything/releases/latest/download/understand-anything-viewer.tgz /path/to/project
# → http://127.0.0.1:5173/?token=... 자동 오픈
```

10MB 이상 대용량 그래프는 git-lfs 권장:
```bash
git lfs install
git lfs track ".ua/*.json"
git add .gitattributes .ua/
```

> 데이터 디렉터리 규칙: 신규 프로젝트는 `.ua/`, 기존에 `.understand-anything/`가 있으면 그것을 계속 사용한다.

---

## 4. 플러그인 vs 스킬 vs MCP

**결론: 플러그인이다. 그 안에 스킬 + 에이전트 + 훅이 들어있는 구조이며, MCP 서버는 사용하지 않는다.**

```
Claude Code Plugin  "understand-anything" v2.9.7   ← 배포 단위
   ├── Skills (9)    슬래시 커맨드. 마크다운 작업 지시서
   ├── Agents (10)   서브에이전트. 격리된 컨텍스트에서 병렬 실행
   ├── Hooks (2)     SessionStart(stale 감지) / PostToolUse(자동 갱신)
   └── Packages (3)  core(엔진) / dashboard(React) / viewer(독립 실행)
```

| 구분 | 정체 | 이 프로젝트 |
|---|---|---|
| **Skill** | LLM에게 주는 마크다운 지시서, 슬래시 커맨드로 호출 | 9개 (`skills/*/SKILL.md`) |
| **Plugin** | Skill + Agent + Hook + 코드를 묶어 배포하는 패키지 | 배포 형태 (`.claude-plugin/plugin.json`) |
| **MCP** | 별도 서버 프로세스, JSON-RPC로 도구 노출 | **사용 안 함** |

### 왜 MCP가 아닌가

- 무거운 작업(파싱, 배치 병합)은 `.mjs` / `.py` 스크립트를 **Bash로 실행**하고, 결과를 **디스크(`.ua/intermediate/`)에 저장**
- LLM 컨텍스트에는 요약만 올려 **토큰 절약**
- 상주 서버가 없어 **설치가 가볍고 플랫폼 이식성이 높음** (17개 플랫폼 지원의 비결)

> 설계 철학: **무거운 건 디스크로, 가벼운 건 컨텍스트로.**

---

## 5. API 토큰이 필요한가

| 상황 | 토큰 | 설명 |
|---|---|---|
| `/understand` 분석 생성 | **간접 필요** | 호스트 도구(Claude Code / Cursor / Codex)의 LLM을 사용. 별도 키 입력은 없고 기존 구독/플랜 토큰을 소모 |
| `/understand-dashboard` | 불필요 | 로컬 JSON 렌더링만 |
| `npx viewer` (팀원 열람) | 불필요 | LLM 호출 0회, Node.js만 필요 |
| `/understand-figma` | **직접 필요** | `FIGMA_TOKEN` 환경변수 필수. 유일하게 외부 API(api.figma.com) 호출 |

### 보안 관점 (소스 확인 결과)

1. **`/understand`는 완전 오프라인** — 외부 네트워크 호출 없음. 코드가 외부 서비스로 전송되지 않는다.
2. **Figma 토큰 취급이 엄격** — 환경변수에서만 읽고 `X-Figma-Token` 헤더로만 전송. 그래프/`meta.json`/로그/중간 파일에 절대 기록하지 않도록 스킬에 명시.
3. **대시보드 파일 접근 이중 보호** — dev 서버 `/file-content.json` 엔드포인트는 **액세스 토큰 + 그래프에서 파생된 경로 허용목록(allowlist)** 으로 제한 (`TokenGate.tsx`). 임의 파일 읽기 차단.

### 비용 절감

- **로컬 모델 사용 가능** — Ollama 등을 호스트 플랫폼의 모델 제공자로 지정하면 API 비용 0원 + 코드 외부 유출 없음 (기업/폐쇄망 환경에 적합)
- 초기 전체 분석만 큰 모델로, 이후 증분 갱신은 저렴한 모델로 운용

---

## 6. 왜 GitHub에서 유명한가

### 객관적 지표

- **Trendshift 등재** (repository #23482)
- 461 커밋 / 62 기여자
- README 8개 언어 (영·중간·중번·일·한·스페인·터키·러시아)
- Better Stack 등 커뮤니티 유튜브 리뷰 생성

### 인기 요인 7가지

1. **명확한 페인포인트** — *"You just joined a new team. The codebase is 200,000 lines of code. Where do you even start?"* 개발자 전원이 공감하는 고통을 정확히 겨냥.
2. **타이밍** — Claude Code / Cursor 플러그인 생태계 초기에 완성도 높은 플러그인으로 선점.
3. **환각 문제의 구조적 해결** — tree-sitter로 구조를 고정해 신뢰성 확보. 경쟁 도구 대비 기술적 차별화가 명확.
4. **시각적 임팩트** — 다크 럭셔리 테마(#0a0a0a + 골드 #d4a574 + DM Serif Display). 스크린샷 한 장이 SNS에서 퍼지기 좋다. 라이브 데모까지 제공.
5. **17개 플랫폼 지원** — Claude 전용이 아니어서 잠재 사용자 풀이 수십 배.
6. **그래프 커밋 → 팀 공유 모델** — 한 명만 토큰을 쓰고 팀 전체가 무료로 혜택. 조직 도입 장벽이 낮다.
7. **엔지니어링 품질** — 증분 분석, 5병렬 배치, 스키마 검증, 무결성 리뷰 에이전트, 66개 테스트, git worktree 같은 엣지 케이스 대응(issue #133). 기여하고 싶어지는 코드베이스.

---

## 7. 로컬 에이전트 구축에 주는 도움

이 저장소는 단순한 도구가 아니라 **프로덕션급 멀티 에이전트 시스템 레퍼런스**다.

### 패턴 1 — 오케스트레이터-워커 + 디스크 버퍼 (가장 중요)

```
메인 스킬(오케스트레이터)이 Phase 0~7을 지휘
  → 각 Phase마다 전문 에이전트 스폰
  → 에이전트는 격리된 컨텍스트에서 작업
  → 결과는 컨텍스트가 아니라 디스크(.ua/intermediate/)에 저장
```

서브에이전트 결과를 메인 컨텍스트로 반환하면 컨텍스트가 폭발한다. **디스크를 중간 버퍼로 쓰면 무한 확장이 가능**하다.

### 패턴 2 — 결정론 / 확률론 분리

tree-sitter(결정론)와 LLM(확률론)의 경계를 어디에 그을지에 대한 실전 기준.

### 패턴 3 — 증분 처리 & 상태 관리

- `fingerprint.ts` — 파일 해시 기반 변경 감지
- `change-classifier.ts` — 변경 유형 분류
- `staleness.ts` — 그래프 낡음 판정
- `meta.json`의 `gitCommitHash` — 마지막 분석 시점 기록

"매번 전부 다시 하지 않는 에이전트"를 만드는 방법.

### 패턴 4 — 병렬성 제어

최대 5 동시 워커, 배치당 20~30파일. 레이트리밋과 처리량의 실전 균형점.

### 패턴 5 — 훅 기반 자동화 (`hooks/hooks.json`)

- `SessionStart` — 그래프 stale 감지 → LLM에게 자동 갱신 지시
- `PostToolUse` (matcher: Bash) — 커밋 감지 → 증분 업데이트 트리거

사용자가 요청하지 않아도 최신 상태를 유지하는 에이전트 설계.

### 패턴 6 — 프롬프트 엔지니어링 교과서

`file-analyzer.md`(529줄), `architecture-analyzer.md`(481줄), `tour-builder.md`(379줄) 등 총 4,092줄의 구조화 출력 강제 프롬프트 실전 예제.

### 패턴 7 — 스키마 우선 설계

`schema.ts` + Zod로 LLM 출력을 검증해 깨진 출력을 조기 차단.

### 바로 재활용 가능한 자산

1. `packages/core` — 43개 언어 파서 엔진을 그대로 라이브러리로 사용 (MIT)
2. `knowledge-graph.json` — 로컬 RAG의 컨텍스트 소스
3. `agents/*.md` — 자신의 도메인에 맞게 개조할 프롬프트 템플릿
4. `.ua/intermediate/` 패턴 — 에이전트 메모리 아키텍처 설계 참고

---

## 8. React / PHP로 만들 수 있는가

**결론: 부분적으로 가능, 전체는 재설계 필요.**

| 계층 | React | PHP | 비고 |
|---|:---:|:---:|---|
| 대시보드 UI | **이미 React** | 비권장 | React 19 + React Flow + Zustand + Tailwind v4 |
| 파서 엔진 | 가능(느림) | 가능 | tree-sitter는 WASM이라 다중 바인딩 존재 |
| 에이전트 오케스트레이션 | 불가 | 가능 | 브라우저는 파일시스템 접근 제약 → 서버사이드 필수 |
| LLM 호출 | 위험 | 적합 | 프론트에 API 키를 두면 탈취 위험 |

### React로 접근할 때

**가능한 것**
- 대시보드는 이미 React. `packages/dashboard/src/`의 컴포넌트 32개(GraphView, NodeInfo, CodeViewer, SearchBar, FileExplorer 등)를 그대로 개조 가능
- `knowledge-graph.json` 스키마만 이해하면 완전히 새로운 UI를 작성 가능
- `@understand-anything/core`의 `./search`, `./types`, `./schema` 서브패스는 브라우저 안전 (메인 엔트리는 Node 모듈을 끌어오므로 import 금지)

**불가능한 것**
- 브라우저만으로 로컬 대규모 파일 스캔
- 프론트엔드에서의 안전한 API 키 보관

**현실적 구성**
```
React 프론트 (그래프 시각화)  ⇄  Node.js 백엔드 (tree-sitter 파싱 + LLM 호출 + 파일 스캔)
```

### PHP로 접근할 때

**유리한 점**
- `tree-sitter-php` 바인딩 존재, PHP 8.3+ 에서 LLM API 호출은 curl로 단순
- Laravel + Queue(Horizon) 조합이면 5병렬 배치 처리가 자연스럽게 구현됨
- 웹 SaaS 배포에 유리
- 한국 시장의 레거시 PHP 코드베이스 분석 수요가 큼

**불리한 점**
- tree-sitter 네이티브 바인딩 설치 난이도 → **파싱만 Node 서브프로세스로 위임**(`shell_exec('node parse.mjs')`)하는 하이브리드 권장
- 장시간 실행 프로세스에 약함 → 큐 시스템 필수

**현실적 구성**
```
Laravel (API + 큐 오케스트레이션 + LLM 호출 + 인증/과금)
   ↓ shell_exec
Node.js 파서 워커 (tree-sitter 파싱 전담)
   ↓ JSON
React 대시보드 (SPA)
```

### 권장 로드맵

| 단계 | 기간 | 내용 | 난이도 |
|---|---|---|---|
| Phase 1 | 1~2주 | 기존 도구로 그래프만 생성 + **React 대시보드를 자체 제작** | 낮음 |
| Phase 2 | 1~2개월 | Laravel/Node 백엔드 추가 → **웹 SaaS**("깃허브 URL 입력 → 그래프") | 중간 |
| Phase 3 | 3개월+ | 자체 파이프라인 + 특정 언어/프레임워크 특화 분석기 | 높음 |

> 전체를 처음부터 만들지 말 것. MIT이므로 core 엔진은 그대로 쓰고 **차별점(UI / 도메인 특화 / 수익 모델)** 에 집중하는 편이 훨씬 빠르다.

---

## 9. 수익화 아이디어 7선

### 라이선스 확인

```
MIT License © Yuxiang Lin and Infinite Universe, Inc.
```
- 상업적 이용 / 수정 / 재배포 / 비공개 소스화 **모두 허용**
- 의무: 저작권 표시 + 라이선스 전문 포함 (About 페이지·README 하단 등)

---

### 아이디어 1 — 엔터프라이즈 온프레미스 SaaS

**문제** 대기업/금융/공공은 코드를 외부 AI에 보낼 수 없지만 레거시 구조를 아는 사람이 없다.

**솔루션** 오픈소스 엔진 + 사내 LLM(Ollama/sLLM) 연동 + SSO/RBAC + 팀 대시보드 + 감사 로그 → 폐쇄망 설치형 제품

| 플랜 | 가격 | 대상 |
|---|---|---|
| Team | 월 30만원 (10석) | 스타트업 |
| Business | 월 150만원 (50석) | 중견기업 |
| Enterprise | 연 3,000만~1억원 | 대기업/금융/공공 |
| 구축 컨설팅 | 건당 1,000~5,000만원 | SI 프로젝트 |

- 수익 잠재력 ★★★★★ / 난이도 ★★★★ / 기간 3~6개월
- 핵심: **폐쇄망 + 한국어 + 온프레미스 = 글로벌 경쟁자가 진입하기 어려운 해자**

---

### 아이디어 2 — "GitHub URL 입력" 웹 SaaS

**컨셉** 설치 불필요. URL 붙여넣기 → 3분 뒤 인터랙티브 지식 그래프 + 공유 링크.

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | 0원 | 공개 레포 3개/월, 그래프 공개 |
| Pro | $19/월 | 비공개 레포 무제한, 비공개 그래프, 증분 갱신 |
| Team | $49/유저/월 | 팀 워크스페이스, PR 임팩트 자동 코멘트, SSO |
| PAYG | $5/분석 | 비정기 사용자 |

부가 수익
- GitHub App 연동 → PR마다 "변경 영향 범위" 자동 코멘트 (유료화 적합)
- README 뱃지 서비스 → 오픈소스 바이럴

- 수익 잠재력 ★★★★ / 난이도 ★★★ / 기간 2~3개월
- 리스크: LLM 토큰이 변동비 → **무료 티어 설계가 수익성을 좌우**

---

### 아이디어 3 — 기술 실사(Tech Due Diligence) 리포트 서비스 ★1순위 추천

**문제** VC 투자·M&A 시 "이 회사 코드 상태가 어떤가"를 빠르게 판단할 방법이 없다. 현재는 시니어 개발자가 2주간 수기 분석.

**솔루션** 코드베이스 → 자동 분석 → 30페이지 PDF 리포트
- 아키텍처 건전성 점수
- 기술 부채 히트맵 (`complexity: complex` 노드 분포)
- 순환 의존성 / 신(God) 오브젝트 탐지
- 테스트 커버리지 구조 (`tested_by` 엣지 활용)
- 버스 팩터(특정 파일 집중도)
- 보안 취약 구조 지점

38종 엣지 타입이 정량 지표 산출의 근거가 된다.

**가격** 건당 500만~2,000만원 (VC/PE) · 리테이너 월 300만원(포트폴리오사 정기 모니터링)

- 수익 잠재력 ★★★★★ / 난이도 ★★ / 기간 1개월
- **1인 창업자에게 최적** — 제품 개발 최소, 초기 자본 거의 0, 즉시 현금 흐름

---

### 아이디어 4 — 한국 시장 로컬라이제이션 + 교육

**근거** 한국은 레거시 Java/Spring·PHP 코드베이스 비중이 높고, 이 도구는 한국어 출력(`--language ko`)을 이미 지원하지만 **한국어 문서·튜토리얼·커뮤니티가 없다.**

| 항목 | 가격 |
|---|---|
| 온라인 강의 "AI로 레거시 코드 정복하기" | 12~20만원 × 수강생 |
| 기업 사내 교육 (1일 워크샵) | 300~500만원/회 |
| 신입 온보딩 자동화 컨설팅 | 1,000만~3,000만원 |
| 유료 뉴스레터/멤버십 | 월 1~3만원 |

`/understand-onboard`의 "온보딩 3개월 → 2주" ROI 스토리가 기업 영업의 핵심 무기.

- 수익 잠재력 ★★★ / 난이도 ★ (가장 쉬움) / 기간 2주~1개월

---

### 아이디어 5 — 도메인 특화 확장 플러그인 판매

코어는 오픈소스로 두고 특정 도메인 분석기를 유료 애드온으로 판매. `plugins/parsers`, `plugins/extractors` 확장 구조가 이미 준비돼 있다.

| 애드온 | 대상 | 가격 |
|---|---|---|
| Legacy Modernizer | Java 8→17 / Spring 레거시 마이그레이션 경로 산출 | $299 일회성 |
| Compliance Mapper | PCI-DSS / HIPAA 관련 코드 경로 추적 | $999/년 |
| Security Graph | 인증·인가 흐름 + 권한 우회 경로 탐지 | $499/년 |
| Solidity/Web3 Analyzer | 스마트컨트랙트 호출 그래프 + 재진입 위험 | $799/년 |
| Cost Graph | 인프라 노드(Terraform/K8s) ↔ 클라우드 비용 매핑 | $399/년 |

- 수익 잠재력 ★★★ / 난이도 ★★★ / 기간 1~2개월/개

---

### 아이디어 6 — 지식 그래프 마켓플레이스

유명 오픈소스(React, Kubernetes, Linux 커널, Next.js 등) 100개의 분석 완료 그래프를 판매. 기여를 시작하려는 개발자가 대상.

- 그래프 1개 $9~29, 전체 열람 구독 $15/월
- 재단/기업 스폰서십 $2,000~10,000

- 수익 잠재력 ★★ / 난이도 ★★ / 기간 1개월
- 리스크: 개발자는 무료를 선호 → 수요 검증 선행 필요

---

### 아이디어 7 — AI 에이전트용 컨텍스트 서버 (장기 / 최대 잠재력)

**통찰** 현재 AI 코딩 에이전트의 최대 병목은 컨텍스트다. `knowledge-graph.json`은 **AI가 읽기 좋은 코드베이스 맵**이다.

```
MCP 서버로 제공:
  query_graph("결제 관련 파일 전부")
    → 관련 노드 + 엣지만 정밀 반환
    → 에이전트가 전체 레포를 읽지 않고도 정확히 작업
= AI 에이전트를 위한 코드베이스 RAG 인프라
```

- API 과금 (쿼리 1만건당 $10), 엔터프라이즈 라이선스 연 $50,000+, 타 AI 툴 회사에 B2B 인프라 납품
- 수익 잠재력 ★★★★★ / 난이도 ★★★★★ / 기간 6개월+

---

### 최종 비교 & 추천 전략

| # | 아이디어 | 수익 | 난이도 | 기간 | 순위 |
|---|---|:---:|:---:|:---:|:---:|
| 3 | 기술 실사 리포트 | ★★★★★ | ★★ | 1개월 | 1 |
| 4 | 한국 교육/컨설팅 | ★★★ | ★ | 2주 | 2 |
| 2 | 웹 SaaS | ★★★★ | ★★★ | 2~3개월 | 3 |
| 1 | 엔터프라이즈 온프레미스 | ★★★★★ | ★★★★ | 3~6개월 | 4 |
| 5 | 특화 애드온 | ★★★ | ★★★ | 1~2개월 | 5 |
| 7 | AI 컨텍스트 서버 | ★★★★★ | ★★★★★ | 6개월+ | 장기 |
| 6 | 그래프 마켓플레이스 | ★★ | ★★ | 1개월 | 실험 |

**3단계 전략**

1. **0~1개월** — 기술 실사 리포트(#3)로 제품 개발 없이 현금 확보 + 고객 니즈 학습
2. **1~3개월** — 그 경험으로 웹 SaaS(#2) MVP 구축. React 대시보드가 이미 있으므로 백엔드만 추가. Step 1 고객이 첫 유료 사용자
3. **3~6개월** — 검증되면 엔터프라이즈(#1)로 확장하거나 #7 인프라로 피벗

**리스크 관리**

1. 원본 오픈소스가 계속 발전 → 무료 버전이 유료 기능을 흡수할 수 있음. **차별점은 코드가 아니라 서비스·도메인·고객 관계에 둘 것**
2. LLM 토큰 = 변동비 → 캐싱·증분 전략이 곧 마진
3. LLM 제공사(Anthropic/OpenAI)의 직접 진입 가능성 → 니치(한국 시장, 특정 산업, 온프레미스)로 방어

---

## 10. 참고 링크

| 항목 | URL |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/Understand-Anything |
| 원본 저장소 (업스트림) | https://github.com/Egonex-AI/Understand-Anything |
| 공식 홈페이지 | https://understand-anything.com |
| 라이브 데모 | https://understand-anything.com/demo/ |
| 한국어 README | https://github.com/Egonex-AI/Understand-Anything/blob/main/READMEs/README.ko-KR.md |
| 라이선스 (MIT) | https://github.com/Egonex-AI/Understand-Anything/blob/main/LICENSE |
| 이슈 트래커 | https://github.com/Egonex-AI/Understand-Anything/issues |
| 독립 뷰어 릴리스 | https://github.com/Egonex-AI/Understand-Anything/releases/latest |
| 원작자 Lum1104 | https://github.com/Lum1104 |
| 관리 조직 Egonex | https://github.com/Egonex-AI |
| Understand Anyone (자매 프로젝트) | https://egonex.ai |
| 커뮤니티 워크스루 (Better Stack) | https://www.youtube.com/watch?v=VmIUXVlt7_I |
| Claude Code 플러그인 문서 | https://code.claude.com/docs/en/plugins-reference |
| Karpathy LLM 위키 패턴 | https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f |
| 그래프 커밋 예시 레포 | https://github.com/GoogleCloudPlatform/microservices-demo |

---

*이 문서는 `bmshin94/Understand-Anything` 저장소를 전수조사하여 작성한 분석 리포트입니다.*
