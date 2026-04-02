# Claude Code 소스 코드 분석서

> 약 1,884개 TypeScript/React 파일을 분석한 심층 기술 문서 — 출처: [wikidocs.net/338204](https://wikidocs.net/338204)

---

## 요약: 세 문장으로 이해하기

Claude Code는 **사용자의 자연어 요청을 받아 AI가 어떤 도구를 써야 하는지 판단하고, 권한 확인을 거쳐 안전하게 실행한 후, 결과를 다시 AI에게 돌려주는 루프**를 핵심으로 한다. 이 루프를 감싸는 시스템들(인증, 설정, 상태 관리, MCP, 플러그인, 브리지, 코디네이터)이 다양한 환경과 사용 사례를 지원한다. 모든 설계 결정은 **안전성**(위험한 작업 차단), **성능**(스트리밍, 병렬 실행, 캐싱), **확장성**(도구, 스킬, 플러그인, MCP)의 세 가지 원칙을 따른다.

---

## 목차

1. [Claude Code란 무엇인가](#1-claude-code란-무엇인가)
2. [전체 구조를 한눈에 보기](#2-전체-구조를-한눈에-보기)
3. [실행 모드](#3-실행-모드)
4. [시작과 초기화 과정](#4-시작과-초기화-과정)
5. [쿼리 루프 — 핵심 엔진](#5-쿼리-루프--핵심-엔진)
6. [QueryEngine — SDK용 래퍼](#6-queryengine--sdk용-래퍼)
7. [도구(Tool) 시스템](#7-도구tool-시스템)
8. [도구 실행 오케스트레이션](#8-도구-실행-오케스트레이션)
9. [명령어(Command) 시스템](#9-명령어command-시스템)
10. [태스크(Task) 시스템](#10-태스크task-시스템)
11. [상태 관리](#11-상태-관리)
12. [서비스 레이어](#12-서비스-레이어)
13. [권한(Permission) 시스템](#13-권한permission-시스템)
14. [훅(Hook) 시스템](#14-훅hook-시스템)
15. [스킬(Skill)과 플러그인(Plugin) 시스템](#15-스킬skill과-플러그인plugin-시스템)
16. [UI 레이어](#16-ui-레이어)
17. [브리지(Bridge) 시스템](#17-브리지bridge-시스템)
18. [원격(Remote) 세션 관리](#18-원격remote-세션-관리)
19. [코디네이터(Coordinator) 모드](#19-코디네이터coordinator-모드)
20. [메모리(Memory) 시스템](#20-메모리memory-시스템)
21. [타입 시스템과 상수](#21-타입-시스템과-상수)
22. [유틸리티 모듈](#22-유틸리티-모듈)
23. [핵심 설계 패턴](#23-핵심-설계-패턴)
24. [전체 데이터 흐름 요약](#24-전체-데이터-흐름-요약)

---

## 1. Claude Code란 무엇인가

Claude Code는 Anthropic이 만든 공식 CLI(터미널 명령줄) 도구다. 사용자가 터미널에서 Claude AI와 대화하면서 코드를 읽고, 수정하고, 실행할 수 있게 해준다. 단순한 챗봇이 아니라, 파일을 직접 편집하고, 셸 명령을 실행하고, 웹을 검색하고, 외부 서비스와 연동하는 "AI 소프트웨어 엔지니어"에 가깝다.

비유하자면, 일반적인 AI 챗봇이 "전화 상담원"이라면, Claude Code는 "현장에 직접 와서 일하는 엔지니어"다. 상담원은 조언만 줄 수 있지만, 현장 엔지니어는 실제로 장비를 만지고 문제를 고칠 수 있다. Claude Code가 바로 그런 존재다 — 사용자의 컴퓨터에서 직접 파일을 열어보고, 코드를 수정하고, 테스트를 실행한다.

기술적으로는 TypeScript로 작성되었고, 터미널 UI는 React 기반의 TUI(Terminal User Interface) 프레임워크인 Ink를 사용한다. 상태 관리는 Zustand 라이브러리를 채택했으며, 빌드 시스템은 bun을 사용해서 사용하지 않는 코드를 자동으로 제거(tree-shaking)한다.

---

## 2. 전체 구조를 한눈에 보기

Claude Code의 동작은 크게 네 단계로 요약할 수 있다. 이 네 단계를 이해하면 1,884개 파일 각각이 어디에 속하는지 자연스럽게 파악할 수 있다.

```
+---------------------+
|   User types in     |
|   the terminal      |
+---------+-----------+
          |
          v
+---------------------+     Phase 1: STARTUP
|     main.tsx        |     - Auth (OAuth / API key / Bedrock / Vertex)
|  (initialization)   |     - Model selection
|                     |     - Load settings & feature gates
+---------+-----------+     - Collect Git status + CLAUDE.md
          |
          v
+---------------------+     Phase 2: QUERY LOOP
|     query.ts        |     - Send messages to Claude API (streaming)
|   (core engine)     |<-+  - Receive response token by token
|                     |  |  - Detect tool_use blocks in response
+---------+-----------+  |
          |              |
          v              |
+---------------------+  |  Phase 3: TOOL EXECUTION
|   Tool Pipeline     |  |  - Validate input (Zod schema)
|  (45+ built-in      |  |  - Check permissions (rules + classifier)
|   tools + MCP)      |  |  - Execute tool (Bash, Read, Edit, ...)
+---------+-----------+  |  - Return result to API
          |              |
          +--------------+  (loop until no more tool_use)
          |
          v
+---------------------+     Phase 4: DISPLAY
|   Ink TUI (React)   |     - Render messages, diffs, progress
|  terminal display   |     - Show tool results
+---------------------+     - Await next user input
```

핵심은 **2단계와 3단계 사이의 루프**다. 사용자가 "이 버그를 고쳐줘"라고 한 번 입력하면, Claude Code는 내부적으로 파일을 읽고 → AI가 분석하고 → 파일을 수정하고 → AI가 검증하고 → 테스트를 실행하는 과정을 여러 턴에 걸쳐 자동으로 반복한다.

---

## 3. 실행 모드

Claude Code는 하나의 프로그램이지만, 상황에 따라 여러 모드로 동작한다. 핵심 엔진(`query.ts`)은 동일하지만, 입력과 출력을 처리하는 방식이 달라진다.

```
+------------------+
|   Claude Code    |
+--------+---------+
         |
+--------+--------+-----------+----------+----------+
|        |        |           |          |          |
v        v        v           v          v          v
+------+ +------+ +----------+ +-------+ +--------+ +--------+
| REPL | |Head- | |Coordin-  | |Bridge | |Kairos  | |Daemon  |
| Mode | |less  | |ator Mode | | Mode  | | Mode   | | Mode   |
+------+ +------+ +----------+ +-------+ +--------+ +--------+
Inter-   No UI,   Leader +     Local     Always-on  Background
active   SDK/pipe Workers      CLI<>Cloud assistant  service
```

| 모드 | 설명 |
|------|------|
| **REPL** | 가장 일반적인 대화형 모드. 터미널에 React 기반 UI 렌더링 |
| **헤드리스** | UI 없이 SDK/파이프라인에서 프로그래밍 방식으로 실행 |
| **코디네이터** | 하나의 리더 에이전트가 여러 워커 에이전트를 관리하는 멀티에이전트 모드 |
| **브리지** | 로컬 터미널을 클라우드 claude.ai 웹 인터페이스와 연결 |
| **어시스턴트(Kairos)** | 상시 대기하는 프로액티브 어시스턴트 모드 |
| **데몬** | 백그라운드에서 실행되는 모드 |
| **뷰어** | 원격 세션을 읽기 전용으로 관찰하는 모드 |

어떤 모드를 사용하든 내부의 핵심 엔진은 동일하다. 차이점은 "사용자 입력을 어디서 받느냐"와 "결과를 어디에 보여주느냐"에 있다.

---

## 4. 시작과 초기화 과정

### 4.1 main.tsx — 모든 것의 출발점

`main.tsx`는 약 800KB에 달하는 거대한 단일 파일이다. 여러 파일로 나누지 않고 하나로 합친 이유는 시작 성능 때문이다 — 하나의 파일은 한 번의 디스크 읽기로 끝나므로 CLI 도구에서 중요한 시작 시간을 단축한다.

```
main.tsx startup sequence
=========================

[1] Parallel I/O prefetch            (saves ~65ms)
    |-- MDM subprocess read
    |-- macOS Keychain prefetch
    |                                   These run WHILE
    v                                   heavy imports load (~135ms)
[2] Conditional module loading
    |-- feature('COORDINATOR_MODE') --> load or skip
    |-- feature('KAIROS')           --> load or skip
    |                                   Dead code eliminated at build time
    v
[3] Early settings load
    |-- Parse --settings CLI flag
    |-- --bare flag? --> skip CLAUDE.md, skills, hooks
    |
    v
[4] Authentication
    |-- Try OAuth token              (claude.ai subscribers)
    |-- Try API key                  (ANTHROPIC_API_KEY)
    |-- Try AWS Bedrock              (sts:AssumeRole)
    |-- Try Google Vertex AI         (GoogleAuth)
    |-- Try Azure Foundry            (DefaultAzureCredential)
    |
    v
[5] Model resolution
    |-- User-specified model?  --> use it
    |-- Subscription tier?     --> Max/Team Premium = Opus
    |                              Others = Sonnet
    v
[6] Build initial state --> Launch REPL or Headless mode
```

**1단계: 병렬 I/O 사전 실행.** 무거운 모듈 임포트가 진행되는 약 135ms 동안 MDM 설정 읽기와 macOS 키체인 인증 토큰 사전 로딩을 병렬로 시작해 약 65ms를 절약한다.

**2단계: 조건부 모듈 로딩.** 피처 게이트 메커니즘으로 비활성화된 기능의 코드는 빌드 결과물에서 아예 제거된다.

**3단계~5단계:** 설정 조기 로딩 → 5가지 인증 방식 순차 시도 → 구독 등급 기반 모델 해석.

**6단계:** 초기 상태 객체 구성 후 선택된 모드로 진입.

### 4.2 컨텍스트 수집

모든 대화에는 두 가지 컨텍스트가 자동 주입된다.

```
Context injection (memoized for session lifetime)
==================================================

System Context                    User Context
+--------------------------+      +--------------------------+
| Git branch: main         |      | CLAUDE.md files          |
| Default branch: main     |      | (project instructions)   |
| Git status (max 2000ch)  |      |                          |
| Recent commits           |      | Today's date             |
| Git user name            |      |                          |
+--------------------------+      +--------------------------+
        |                                  |
        +----------------+-----------------+
                         |
                         v
                Injected into every
                API conversation turn
```

**시스템 컨텍스트**는 현재 Git 브랜치, 기본 브랜치, Git 상태, 최근 커밋, Git 사용자 이름을 포함한다. 병렬로 수집되며 Git 상태가 너무 길면 2,000자에서 잘라낸다. 한 번 수집되면 세션이 끝날 때까지 캐시된다.

**사용자 컨텍스트**는 프로젝트의 CLAUDE.md 파일들과 오늘 날짜를 포함한다.

---

## 5. 쿼리 루프 — 핵심 엔진

### 5.1 기본 구조

`query.ts`(약 68KB)는 Claude Code의 심장이다. "비동기 제너레이터" 패턴을 사용해 API 응답이 글자 단위로 도착하는 즉시 화면에 표시한다 — 사용자는 AI가 "타이핑하는 것"을 실시간으로 볼 수 있다.

### 5.2 턴(Turn)의 처리 흐름

```
Query Loop: one turn
=====================

+--[1. Message Preprocessing]-------------------------------+
|                                                           |
|  Snip Compact -----> remove old messages entirely         |
|  Microcompact -----> shrink tool_use blocks inline        |
|  Context Collapse -> stage collapse operations            |
|  Auto-Compact -----> summarize if near token limit        |
|                      (threshold = context_window - 13000) |
+----------------------------+------------------------------+
                             |
                             v
+--[2. API Streaming Call]----------------------------------+
|                                                           |
|  Send: messages + system prompt + tools schema            |
|  Receive: streaming response (token by token)             |
|                                                           |
|  If model overloaded --> switch to fallback model          |
+----------------------------+------------------------------+
                             |
                             v
+--[3. Error Withholding & Recovery]-----------------------+
|                                                           |
|  413 Prompt Too Long:                                     |
|    try collapse drain --> try reactive compact --> fail    |
|                                                           |
|  Max Output Tokens:                                       |
|    escalate 8K->64K --> retry up to 3 times --> fail      |
+----------------------------+------------------------------+
                             |
                             v
+--[4. Tool Execution]--------------------------------------+
|                                                           |
|  Safe tools (Read, Grep, Glob):                           |
|    --> run up to 10 in parallel                           |
|                                                           |
|  Unsafe tools (Edit, Bash, Write):                        |
|    --> run one at a time, sequentially                    |
|                                                           |
|  Large results --> persist to disk, pass reference        |
+----------------------------+------------------------------+
                             |
                             v
+--[5. Post-processing]-------------------------------------+
|                                                           |
|  Run Stop Hooks (validation)                              |
|  Check token budget                                       |
|  Check max turns limit                                    |
|                                                           |
|  tool_use in response?                                    |
|    YES --> build new State, go back to step 1             |
|    NO  --> exit loop, return to user                      |
+-----------------------------------------------------------+
```

**실제 예시:** 사용자가 "Fix the bug in auth.ts"라고 입력하면 내부에서는 이런 일이 벌어진다.

```
Turn 1:  AI: "Let me read the file first."
         --> tool_use: FileRead("auth.ts")
         --> result: [file contents returned]

Turn 2:  AI: "I see a null check missing on line 42. Let me fix it."
         --> tool_use: FileEdit("auth.ts", old="user.name", new="user?.name")
         --> result: "Edit applied successfully"

Turn 3:  AI: "Let me verify the fix by running tests."
         --> tool_use: Bash("npm test -- auth.test.ts")
         --> result: "All 12 tests passed"

Turn 4:  AI: "Done! I fixed the null check on line 42."
         --> no tool_use --> loop exits, response shown to user
```

사용자는 처음에 한 번 입력하고 결과만 기다린다. 내부에서는 4번의 턴, 3번의 도구 실행, 4번의 API 호출이 발생했다.

---

## 6. QueryEngine — SDK용 래퍼

QueryEngine은 `query.ts`의 쿼리 루프를 외부 프로그램이 쉽게 사용할 수 있게 감싼 클래스다.

```
External Program (Agent SDK, etc.)
        |
        v
+--[QueryEngine]----------------------------------+
|                                                  |
|  submitMessage("Fix the bug in auth.ts")         |
|      |                                           |
|      +--> Save transcript to disk (crash-safe)   |
|      |                                           |
|      +--> query() generator loop                 |
|      |      |                                    |
|      |      +--> yield SDKMessage (streaming)    |
|      |      +--> yield SDKMessage ...            |
|      |                                           |
|      +--> Accumulate usage (tokens, cost)        |
|      +--> Check budget (maxBudgetUsd)            |
|      +--> yield SDKResult (final)                |
|                                                  |
|  State persisted across turns:                   |
|    - mutableMessages[]                           |
|    - totalUsage                                  |
|    - permissionDenials[]                         |
+--------------------------------------------------+
```

API 호출 중 프로세스가 강제 종료되어도 사용자 메시지까지는 복구할 수 있다. 비용 예산이 설정되어 있으면 누적 비용이 예산을 초과하는 순간 자동으로 중단한다.

---

## 7. 도구(Tool) 시스템

### 7.1 도구란 무엇인가

"도구"는 Claude가 외부 세계와 상호작용하기 위한 수단이다. Claude AI는 그 자체로는 텍스트만 생성할 수 있다. 파일을 읽거나 코드를 실행하는 것은 불가능하다. Claude Code가 "다리" 역할을 한다 — AI가 `FileRead` 도구를 요청하면 Claude Code가 실제로 파일을 읽고 결과를 AI에게 돌려준다.

45개 이상의 내장 도구가 있으며, MCP(Model Context Protocol)를 통해 GitHub, Slack, 데이터베이스 같은 외부 도구도 추가할 수 있다.

### 7.2 도구의 공통 구조

```
Every Tool has:
+-----------------------------------------------------------------+
|  name          "BashTool", "FileEditTool", ...                  |
|  inputSchema   Zod schema defining required/optional inputs     |
|  call()        The actual execution function                    |
|  description() Text shown to AI so it knows when to use it     |
|                                                                 |
|  checkPermissions()   Can this run in current context?          |
|  validateInput()      Is the input semantically valid?          |
|  isConcurrencySafe()  Safe to run alongside other tools?        |
|  isReadOnly()         Does it modify anything?                  |
|  maxResultSizeChars   Over this limit -> persist to disk        |
|  render*()            React components for terminal UI          |
+-----------------------------------------------------------------+
```

### 7.3 도구 등록과 조립

모든 도구는 `tools.ts`에 특정 순서로 등록된다. 순서가 중요한 이유는 API의 프롬프트 캐싱 안정성 때문이다 — 도구 순서가 바뀌면 캐시가 무효화되어 비용이 증가한다.

```
Tool Assembly Pipeline
======================

getAllBaseTools()                     44+ tools registered
      |
      v
Feature gate filter                  Remove disabled features
      |
      v
Deny rules filter                    Remove user-blocked tools
      |
      v
Mode filter                          SIMPLE mode: only Bash, Read, Edit
      |
      v
Built-in tools (sorted by name)
      |
      +------> assembleToolPool() <------+
                      |                   |
                      v              MCP tools
               Merged tool list      (sorted by name)
```

### 7.4 주요 도구 설명

```
Tool Categories
===============

File Operations     Shell          Search          Web
+-------------+  +---------+   +-----------+   +----------+
| FileRead    |  | Bash    |   | Grep      |   | WebFetch |
| FileEdit    |  |         |   | Glob      |   | WebSearch|
| FileWrite   |  |         |   | ToolSearch|   |          |
| NotebookEdit|  |         |   |           |   |          |
+-------------+  +---------+   +-----------+   +----------+

Agent/Team          Planning        Tasks           Skill
+-------------+  +-----------+   +----------+   +----------+
| Agent       |  | EnterPlan |   | TaskCreate|  | Skill    |
| SendMessage |  | ExitPlan  |   | TaskGet   |  |          |
| TeamCreate  |  |           |   | TaskUpdate|  |          |
| TeamDelete  |  |           |   | TaskList  |  |          |
+-------------+  +-----------+   +----------+   +----------+
```

**BashTool** — 셸 명령을 실행한다. Tree-sitter 파서로 명령어의 구조를 분석하고, 허용 목록에 있는 구문만 통과시키는 "기본 거부(fail-closed)" 설계다. 명령 실행이 15초를 초과하면 자동으로 백그라운드 태스크로 전환되고, 2초마다 진행 상황을 보고한다.

**AgentTool** — "다른 AI를 고용하는" 도구다. 서브에이전트는 다섯 가지 방식으로 실행될 수 있다: 같은 프로세스에서 즉시 실행, 백그라운드 비동기 실행, tmux 패널에 별도 팀메이트로 스폰, 격리된 Git 워크트리에서 실행, 또는 클라우드에서 원격 실행. 서브에이전트는 부모보다 제한된 권한을 받는다.

**FileEditTool** — 파일의 특정 문자열을 교체한다. 퍼지 매칭으로 의도한 위치를 찾고, 인코딩과 줄바꿈을 보존하며, Git diff를 생성한다.

**GrepTool** — ripgrep 기반 텍스트 검색. 세 가지 출력 모드(content, files_with_matches, count)와 기본 250개 결과 제한을 지원한다.

---

## 8. 도구 실행 오케스트레이션

### 8.1 실행 파이프라인 10단계

```
Tool Execution Pipeline (for each tool_use block)
==================================================

[1] Lookup tool by name ---> not found? try aliases
              |
[2] Check abort signal ---> user pressed Ctrl+C? exit
              |
[3] Validate input (Zod) -> bad format? friendly error
              |
[4] Run PreToolUse hooks -> hook says block? stop here
              |
[5] Check permissions -----> deny? ask user? auto-classify?
              |
[6] Execute tool.call() ---> the actual work happens here
              |
[7] Map result to API format
              |
[8] Persist if oversized --> save to disk, return reference
              |
[9] Run PostToolUse hooks
              |
[10] Log telemetry event
```

### 8.2 동시성 모델

```
Tool Concurrency: Partitioning Algorithm
=========================================

Input:  [Read] [Grep] [Glob] [Edit] [Read] [Read] [Bash]
         safe   safe   safe  UNSAFE  safe   safe  UNSAFE

Batch 1: [Read, Grep, Glob]  --> parallel (up to 10)
Batch 2: [Edit]              --> serial (alone)
Batch 3: [Read, Read]        --> parallel
Batch 4: [Bash]              --> serial (alone)
```

연속된 동시성 안전 도구들은 최대 10개까지 병렬 실행된다. 비안전 도구를 만나면 새 배치가 시작되고 단독으로 실행된다. "안전한 것은 빠르게 병렬로, 위험한 것은 천천히 하나씩"이라는 원칙이다.

### 8.3 스트리밍 도구 실행기

API 응답이 아직 스트리밍 중일 때부터 이미 도착한 도구 사용 블록의 실행을 시작하는 최적화가 있다.

```
Streaming Tool Executor (overlaps API streaming + execution)
=============================================================

Time --->

API stream:   [...text...][tool_use A][...text...][tool_use B][done]
                              |                        |
Execution:              start A                   start B
                        |..running..|done     |..running..|done
```

---

## 9. 명령어(Command) 시스템

명령어는 사용자가 슬래시(`/`)로 시작하는 입력을 통해 호출하는 기능이다. 도구가 "AI가 사용하는 기능"이라면, 명령어는 "사용자가 직접 사용하는 기능"이다. 80개 이상의 명령어가 있다.

```
Command Types
=============

/commit, /review ...       /settings, /doctor ...    /help ...
+-------------------+      +-------------------+     +-----------+
| Prompt Command    |      | Local Command     |     | Slash Cmd |
| Expands to a      |      | Renders JSX UI    |     | Can be    |
| system prompt for |      | in the terminal   |     | either    |
| the AI to follow  |      |                   |     | type      |
+-------------------+      +-------------------+     +-----------+

Additional sources:
  +-- Plugin commands   (from ~/.claude/plugins/)
  +-- Skill commands    (from ~/.claude/skills/)
  +-- MCP commands      (from connected MCP servers)
```

**프롬프트 명령어**는 AI에게 전달할 프롬프트로 확장된다. 예를 들어 `/commit`은 "현재 변경 사항을 분석하고 커밋 메시지를 작성하라"는 프롬프트로 확장된다. **로컬 명령어**는 React 컴포넌트를 렌더링하는 UI 기반 명령어다.

---

## 10. 태스크(Task) 시스템

태스크는 백그라운드에서 실행되는 비동기 작업을 관리하는 시스템이다.

```
Task Lifecycle
==============

Spawn                  Register               Run in background
(BashTool async,  -->  AppState.tasks[id]  --> output -> disk file
 AgentTool, etc.)      status: "running"       progress reported
                                                     |
                                                     v
                       AI reads output          Completion
                       via TaskOutputTool  <--  status: "completed"
                                                or "failed" / "killed"

Task Types:
  b________ = local_bash          (shell command)
  a________ = local_agent         (in-process sub-agent)
  r________ = remote_agent        (cloud execution)
  t________ = in_process_teammate (team member)
  w________ = local_workflow      (workflow script)
  m________ = monitor_mcp         (MCP server watch)
  d________ = dream               (async continuation)
```

각 태스크는 고유 ID(유형 접두사 + 8자리 랜덤 문자)를 가지며, 출력을 디스크 파일로 리디렉션한다. 상태는 `pending`, `running`, `completed`, `failed`, `killed` 중 하나다.

---

## 11. 상태 관리

### 11.1 AppState — 글로벌 상태

```
AppState (DeepImmutable)
========================

+-- Settings & Config ----+  +-- UI State -----------+
|  settings               |  |  expandedView         |
|  mainLoopModel          |  |  statusLineText       |
|  toolPermissionContext   |  |  spinnerTip           |
+-------------------------+  +------------------------+

+-- Agent / Team ---------+  +-- Tasks ---------------+
|  agentNameRegistry      |  |  tasks (mutable!)      |
|  teamContext            |  |  foregroundedTaskId     |
|  viewingAgentTaskId     |  |                        |
+-------------------------+  +------------------------+

+-- MCP -----------------+   +-- Plugins -------------+
|  clients[]             |   |  enabled[]             |
|  tools[]               |   |  disabled[]            |
|  commands[]            |   |  errors[]              |
|  resources{}           |   |                        |
+------------------------+   +------------------------+

+-- Bridge / Remote -----+   +-- Feature Flags -------+
|  replBridgeConnected   |   |  kairosEnabled         |
|  remoteSessionUrl      |   |  fastMode              |
|  connectionStatus      |   |  effortValue           |
+------------------------+   +------------------------+
```

"불변(Immutable)" 제약이 적용되어 있다 — 상태를 직접 수정할 수 없고, 항상 새로운 객체를 만들어야 한다. 이렇게 하면 "이전 상태"와 "새 상태"를 비교하여 무엇이 변했는지 정확히 알 수 있다.

### 11.2 상태 변경의 부수효과

상태가 변경될 때 자동으로 발생하는 부수효과들:

- 권한 모드 변경 → CCR과 SDK 리스너에게 알림
- 모델 변경 → 설정 파일에 영속화
- 설정 변경 → 인증 캐시 무효화

---

## 12. 서비스 레이어

### 12.1 API 클라이언트와 재시도

API 클라이언트는 Claude API와의 모든 통신을 담당하며, 약 3,000줄에 달한다. 주요 역할:

- **베타 기능 조립**: 모델 능력에 따라 `thinking`, `tool_search` 등 동적 활성화
- **프롬프트 캐싱**: 시스템 프롬프트와 도구 스키마를 1시간 캐시하여 API 비용 절감
- **도구 스키마 정규화**: 내부 Tool 객체를 API JSON 형식으로 변환

```
API Retry Strategy
==================

Error        Action
-----        ------
429          retry-after < 500ms? wait & retry (preserve cache)
(rate limit) otherwise: disable fast mode, switch to standard model

529          3 consecutive? switch to fallback model
(overloaded) non-foreground tasks: give up immediately

401          force-refresh OAuth token, recreate client
(auth fail)

ECONNRESET   disable keep-alive, recreate client
EPIPE

Persistent   retry forever with exponential backoff
mode (ANT)   up to 6 hours total (unattended sessions only)
```

### 12.2 자동 압축 서비스

토큰 사용량이 임계값(유효 윈도우 - 13,000 토큰)을 넘으면 자동으로 작동한다.

```
Auto-Compact Flow
=================

Token usage > threshold?
      |
      v
Circuit breaker check (3 consecutive failures = stop trying)
      |
      v
Try session memory compact first (preserves granularity)
      |
      | failed?
      v
Full conversation compact:
  1. Strip images (save tokens)
  2. Group messages by API round
  3. Generate summary via forked sub-agent
  4. Replace old messages with summary
  5. Restore top 5 referenced files (50K token budget)
  6. Re-inject skills (25K budget, 5K per skill)
```

### 12.3 MCP 프로토콜 서비스

MCP(Model Context Protocol)는 외부 도구와 리소스를 Claude Code에 통합하는 표준 프로토콜이다.

```
MCP Server Connection States
=============================

Connected ----> Failed -----> NeedsAuth
    ^             |               |
    |             v               v
    +------- Pending <-----------+
              (retry: 1s -> 30s exponential backoff, max 5 attempts)

Transport Types:
  Stdio ------> spawn local process
  SSE/HTTP ---> connect to remote server
  WebSocket --> bidirectional communication
  SDK --------> built-in server
  Claude.ai --> proxy relay
```

### 12.4 비용 추적

비용은 메모리 내에서 실시간으로 누적되고, 세션 종료 시 프로젝트 설정에 영속화된다. 모델별로 입력/출력/캐시 토큰이 세분화되어 추적되며, 어드바이저 비용도 재귀적으로 합산된다.

---

## 13. 권한(Permission) 시스템

### 13.1 왜 권한 시스템이 필요한가

Claude Code는 사용자 컴퓨터에서 파일을 수정하고 명령을 실행할 수 있다. 권한 시스템은 "어떤 도구를, 어떤 입력으로, 실행해도 되는가"를 판단하는 게이트키퍼 역할을 한다.

### 13.2 권한 모드와 확인 파이프라인

```
Permission Pipeline
===================

Tool use request arrives
      |
[1]   v
validateInput()            Is the input semantically valid?
      |
[2]   v
checkPermissions()         Tool-specific rules
      |                    (e.g. file path in allowed dirs?)
[3]   v
Run PreToolUse hooks       User-defined hooks can block
      |
[4]   v
Match against rules        +-- alwaysAllow rules --> APPROVE
      |                    +-- alwaysDeny rules  --> DENY
      |                    +-- alwaysAsk rules   --> ASK USER
      |
[5]   v  (no rule matched)
Which permission mode?
      |
      +-- Default mode --> ASK USER (show prompt)
      |
      +-- Auto mode ----> AI Classifier (2-stage)
      |                     Stage 1: Fast (streaming)
      |                     Stage 2: Thinking (deep analysis)
      |                       |
      |                       +-- safe --> APPROVE
      |                       +-- risky -> ASK USER
      |
      +-- Plan mode ----> read-only tools only
      |
      +-- Bypass mode --> APPROVE everything

Rule sources (highest to lowest priority):
  Local settings > Project settings > User settings > Flags > Policy
```

| 모드 | 설명 |
|------|------|
| **Default** | 읽기 전용은 자동 승인, 위험한 작업은 사용자에게 확인 |
| **Auto** | AI 분류기가 2단계(빠른 판단 + 심층 분석)로 위험도 평가 |
| **Plan** | 읽기 전용 도구만 허용하는 계획 수립 전용 모드 |
| **Bypass** | 모든 것을 자동 승인 (개발 환경용) |

---

## 14. 훅(Hook) 시스템

훅은 특정 이벤트가 발생했을 때 자동으로 실행되는 사용자 정의 동작이다.

```
Hook Event Timeline
====================

SessionStart                                        Stop
    |                                                |
    v                                                v
[session begins]                               [AI response done]
    |                                                |
    |   UserPromptSubmit                             |
    |       |                                        |
    v       v                                        |
    |   [user types message]                         |
    |       |                                        |
    |       |   PreToolUse    PostToolUse            |
    |       |       |              |                 |
    v       v       v              v                 v
----+-------+-------+--------------+-----------------+----> time
            |       |              |
            |   [tool executes]    |
            |                      |
            |   PostToolUseFailure (if tool failed)
            |
        PermissionRequest (when permission needed)
        Notification (when alert fires)

Hook Response Controls:
  continue: false  --> stop current operation
  decision: block  --> deny the tool execution
  updatedInput     --> modify tool input before execution
  additionalContext --> inject extra context
```

| 훅 | 시점 | 용도 |
|-----|------|------|
| **PreToolUse** | 도구 실행 직전 | 승인/차단/입력 수정 |
| **PostToolUse** | 실행 직후 | 결과 검증 |
| **UserPromptSubmit** | 사용자 입력 시 | 추가 컨텍스트 주입 |
| **Stop** | 응답 완료 후 | 최종 검증 |

---

## 15. 스킬(Skill)과 플러그인(Plugin) 시스템

### 15.1 스킬

스킬은 재사용 가능한 작업 템플릿이다.

```
Tools vs Skills vs Plugins vs Commands
========================================

Tool       Low-level, atomic action        "Read this file"
           Used by AI automatically         "Run this command"

Skill      High-level, reusable template   "Review this code"
           Invoked by user as /command      "Create a commit"

Plugin     Package of skills + hooks +     "GitHub integration"
           MCP servers bundled together     "Slack integration"

Command    User-facing / shortcut          /help, /settings, /model
           Can be prompt-type or UI-type
```

```
Skill System
============

Disk-based skills                     Bundled skills
(user-created)                        (built into binary)

~/.claude/skills/                     src/skills/bundled/
.claude/skills/                       registerBundledSkill()
     |                                     |
     v                                     v
+----------+    YAML frontmatter:    +----------+
| commit.md|    name, description,   | simplify |
| review.md|    whenToUse, tools,    | commit   |
| ...      |    model, paths         | ...      |
+----------+                         +----------+
     |                                     |
     +------------------+------------------+
                        |
                        v
               Available as /commands
               Full content loaded only on invocation
```

### 15.2 플러그인

플러그인은 스킬, 훅, MCP 서버를 하나의 패키지로 묶은 것이다.

```
Plugin = Skills + Hooks + MCP Servers (bundled together)
=========================================================

BuiltinPlugin
+--------------------------------------------+
|  name: "github-integration"                |
|                                            |
|  skills:     [pr-review, autofix-pr, ...]  |
|  hooks:      { PreToolUse: [...] }         |
|  mcpServers: { github: { command: ... } }  |
|                                            |
|  isAvailable()  --> conditional availability|
|  defaultEnabled --> on/off by default       |
+--------------------------------------------+
```

플러그인의 에러 처리는 20가지 이상의 구체적 유형(일반 에러, 매니페스트 파싱 에러, MCP 설정 오류, LSP 서버 충돌, 정책 차단 등)으로 세분화되어 있다.

---

## 16. UI 레이어

### 16.1 자체 제작 TUI 프레임워크 (Ink)

React의 컴포넌트 모델이 복잡한 터미널 UI에 매우 적합하기 때문에, 기존 터미널 라이브러리 대신 React 기반의 자체 TUI 프레임워크를 사용한다.

```
Ink TUI Rendering Pipeline
===========================

React component update
      |
      v
Reconciler calculates diff  (custom react-reconciler)
      |
      v
Yoga layout engine           (Flexbox for terminal)
calculates positions
      |
      v
Render to screen buffer      Double buffering:
+------------------+         +------------------+
| back frame (new) |  diff   | front frame (old)|
|                  | ------> |                  |
+------------------+  only   +------------------+
                      changed
                      cells
      |
      v
Write ANSI to terminal       Throttled at FRAME_INTERVAL_MS

Memory optimization:
  CharPool ------> intern strings (one copy of "hello")
  StylePool -----> intern ANSI codes + pre-serialized transitions
  HyperlinkPool -> intern URLs
  Dirty tracking > skip unchanged subtrees
```

**핵심 최적화 기법:**

- **이중 버퍼링**: 다음 화면을 완성한 후 변경된 셀만 터미널에 출력해 깜빡임 제거
- **객체 풀링**: 같은 문자열/스타일을 하나만 저장하고 인덱스로 참조
- **더티 추적**: 변경된 부분만 다시 그려 낭비 방지
- **프레임 조절**: 업데이트 빈도 제한으로 터미널 성능 유지

### 16.2 화면 구성

```
REPL Screen Layout
===================

+---------------------------------------------------+
|  Logo Header (memoized, rarely re-renders)         |
+---------------------------------------------------+
|                                                     |
|  Message List (virtualized)                         |
|  +-----------------------------------------------+ |
|  | User: Fix the bug in auth.ts                   | |
|  |                                                | |
|  | Assistant: I'll look at the file...            | |
|  |   [Read] auth.ts                               | |
|  |   [Edit] auth.ts (line 42)                     | |
|  |   Done! Fixed the null check.                  | |
|  +-----------------------------------------------+ |
|                                                     |
+---------------------------------------------------+
|  Task/Teammate Panel (toggle with Ctrl+T)          |
|  [task a3f2k1m9: running] [task b1c4d7e8: done]   |
+---------------------------------------------------+
|  Prompt Input                                       |
|  > mode: prompt | bash                              |
|  > [type here...]           [autocomplete dropdown] |
|  > status bar: model, tokens, cost                  |
+---------------------------------------------------+
```

### 16.3 키바인딩 시스템

```
Keybinding Contexts
====================

Global:       Ctrl+C = interrupt,  Ctrl+D = exit,  Ctrl+T = tasks
Chat:         Enter = submit,  Up/Down = history
Autocomplete: Tab = accept,  Esc = dismiss,  Up/Down = navigate
Transcript:   Ctrl+O = toggle,  q = exit

Chord support:  Ctrl+K -> Ctrl+S  (two-key sequence)

Customizable via ~/.claude/keybindings.json
```

---

## 17. 브리지(Bridge) 시스템

브리지는 로컬 터미널의 Claude Code를 클라우드의 Claude Remote Runtime(CCR)에 연결하는 시스템이다. 33개의 소스 파일로 구성된 복잡한 서브시스템이다.

**왜 브리지가 필요한가?** claude.ai 웹 인터페이스에서 요청한 작업은 사용자의 로컬 컴퓨터에서 실행되어야 한다. 브리지가 이 간극을 메운다 — 클라우드에서 온 작업 요청을 받아서 로컬에서 실행하고, 결과를 다시 클라우드로 보낸다.

```
Bridge Architecture
====================

claude.ai (web)               Local Machine
+----------------+            +---------------------------+
|  User types    |            |  Bridge (bridgeMain.ts)   |
|  in browser    |            |                           |
|       |        |            |  [1] Register environment |
|       v        |   poll     |  [2] Poll for work        |
|  CCR Backend   |<-----------|  [3] Spawn session        |
|  stores work   |            |  [4] ACK work             |
|       |        |            |  [5] Heartbeat loop       |
|       |        |  results   |  [6] Archive on done      |
|       |        |----------->|                           |
+-------+--------+            +---------------------------+
        |                              |
        v                              v
+----------------+            +------------------+
|  Web UI shows  |            |  Child process   |
|  results       |            |  (Claude Code)   |
+----------------+            |  executes locally|
                              +------------------+

Multi-session: up to 32 parallel sessions
Dedup: BoundedUUIDSet (circular buffer, fixed memory)

Token Refresh:
  CCR v1: OAuth token, direct refresh
  CCR v2: Session JWT (~5h55m), reconnect to server for new token
          Pre-scheduled 5 minutes before expiry

Backoff:
  Connection errors: 2s -> 120s (cap), give up after 10 min
  General errors:    500ms -> 30s (cap)
  Shutdown grace:    SIGTERM, then SIGKILL after 30s
```

---

## 18. 원격(Remote) 세션 관리

원격 세션 관리자는 단일 CCR 세션에 WebSocket으로 연결하여 메시지를 스트리밍한다.

```
Remote Session WebSocket
=========================

wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe

State Machine:

closed ---(connect)---> connecting ---(onopen)---> connected
  ^                                                    |
  |                                                    |
  +---------(close/error, max 5 retries)---------------+

Reconnection:
  - General disconnect: max 5 attempts, 2s delay
  - Session not found (4001): 3 retries (transient during compaction)
  - Auth failure (4003): no retry
  - Ping/Pong every 30s to detect stale connections

Permission flow:
  Server: "Can this tool run?" --> Client: show prompt to user
  Client: "Allow" or "Deny"   --> Server: proceed or skip
```

**브리지 vs 원격 세션 관리자:**
- **브리지** = 공항 컨트롤 타워 — 여러 세션의 이착륙을 관리
- **원격 세션 관리자** = 조종석의 통신 장비 — 하나의 세션과 관제탑 사이의 실시간 통신

---

## 19. 코디네이터(Coordinator) 모드

코디네이터 모드는 하나의 "리더" 에이전트가 여러 "워커" 에이전트를 관리하는 멀티에이전트 오케스트레이션 시스템이다.

```
Coordinator Architecture
=========================

                +-------------------+
                |  LEADER (main)    |
                |  - AgentTool      |  Does NOT edit code directly.
                |  - SendMessage    |  Delegates everything.
                |  - TaskStop       |
                +--------+----------+
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
   +------------+ +------------+ +------------+
   | WORKER 1   | | WORKER 2   | | WORKER 3   |
   | QueryEngine| | QueryEngine| | QueryEngine|
   |            | |            | |            |
   | Read, Grep | | Edit, Bash | | Read, Test |
   | WebFetch   | | Write      | | Grep       |
   +------------+ +------------+ +------------+
    (isolated)    (isolated)     (isolated)
    permissions   permissions    permissions

Work Phases:

[1] Research     Multiple workers in parallel
    (parallel)   Each explores different files/angles
        |
[2] Synthesis    Leader reads all results
    (sequential) Leader MUST understand (no delegation)
        |
[3] Implement    Workers modify code
    (per-area)   One area at a time to avoid conflicts
        |
[4] Verify       Workers run tests in parallel
    (parallel)   Independent test suites
```

리더 에이전트는 직접 코드를 수정하지 않는다. 종합 단계에서 리더는 반드시 워커의 결과를 **직접 이해**해야 한다 — "워커가 알아서 했을 테니 넘어가자"는 식의 위임은 금지된다.

---

## 20. 메모리(Memory) 시스템

메모리 시스템은 대화가 끝나도 유지되는 영속적인 정보 저장소다.

```
Memory System Structure
========================

~/.claude/projects/{project-slug}/memory/
|
+-- MEMORY.md              (index file, max 200 lines / 25KB)
|   |
|   +-- "- [User role](user_role.md) -- senior Go dev, new to React"
|   +-- "- [Testing](feedback_testing.md) -- use real DB, not mocks"
|   +-- "- [Merge freeze](project_freeze.md) -- until 2026-03-05"
|   +-- "- [Bug tracker](reference_linear.md) -- Linear INGEST project"
|
+-- user_role.md           (type: user)
+-- feedback_testing.md    (type: feedback)
+-- project_freeze.md      (type: project)
+-- reference_linear.md    (type: reference)
```

| 유형 | 내용 |
|------|------|
| **user** | 사용자의 역할, 전문성, 선호도 |
| **feedback** | 작업 방식 지침 (교정사항 + 확인사항) |
| **project** | 현재 진행 중인 작업, 목표, 기한 |
| **reference** | 외부 시스템 위치 포인터 |

단순히 규칙만 기록하는 게 아니라, "왜(Why)"와 "어떻게 적용(How to apply)"도 함께 기록하여 엣지 케이스에서도 올바르게 판단할 수 있게 한다.

> ⚠️ 저장하지 않아야 할 것: 코드 패턴, 아키텍처, git 히스토리, 디버깅 레시피, CLAUDE.md에 이미 있는 내용

---

## 21. 타입 시스템과 상수

### 21.1 핵심 타입 정의

Claude Code의 타입 시스템은 `types/` 디렉토리에 중앙 집중적으로 정의되어 있다.

- **메시지 타입**: 사용자/어시스턴트/시스템/진행/첨부/도구결과로 구분
- **권한 타입** (62KB): 모드, 규칙 소스, 규칙 값, 결정(허용/거부/질문)이 세밀하게 타입화
- **ID 타입**: `SessionId`와 `AgentId`를 브랜딩 타입으로 정의하여 혼용 방지

### 21.2 설정 소스 우선순위

```
Settings Priority (highest to lowest)
======================================

[1] Local     .claude/settings.local.json    (editable)
[2] Project   .claude/settings.json          (editable)
[3] User      ~/.claude/settings.json        (editable)
[4] Flags     flag file                      (read-only)
[5] Policy    enterprise policy              (read-only)
```

---

## 22. 유틸리티 모듈

**Bash 보안** (888KB, 18파일) — Tree-sitter 파서로 셸 명령어의 AST를 분석한다. 허용 목록에 있는 노드 타입만 통과시키는 "기본 거부(fail-closed)" 설계. 명령어 치환, 변수 확장, 리디렉션을 모두 분석한다.

**샌드박스** — 도구 실행을 격리된 환경에서 수행한다. `//path`는 절대 루트, `/path`는 설정 파일 기준 상대, `~/path`는 홈 디렉토리를 의미한다.

**토큰 계산** — `tokenCountWithEstimation()`이 핵심으로, 마지막 API 응답의 정확한 토큰 수에 이후 메시지의 추정치를 더한다. 자동 압축, 세션 메모리, 임계값 확인 등에 사용된다.

**스웜/팀 관리** — 팀 파일(TeamFile)을 통해 에이전트 간 협업을 관리한다. 인프로세스 러너는 메시지 라우팅, 도구 필터링, 권한 동기화를 처리한다.

**글로벌 세션 상태** (56KB) — 세션 ID, 누적 비용, API 지속시간, 모델별 사용량, 등록된 훅, 에이전트 색상 맵 등을 세션 수명 동안 유지한다.

---

## 23. 핵심 설계 패턴

Claude Code 전반에서 반복적으로 나타나는 설계 패턴 여덟 가지:

```
Design Patterns Summary
========================

[1] Generator Streaming     query() yields events one-by-one
                            --> real-time display of AI "thinking"

[2] Feature Gate            bun:bundle dead-code elimination
    Dead Code Removal       --> disabled features removed at build time

[3] Memoized Context        getSystemContext(), getUserContext()
                            --> computed once, cached for session

[4] Withhold & Recover      Buffer recoverable errors (413, max tokens)
                            --> try auto-fix before showing to user

[5] Lazy Import             Wrap in function to avoid circular deps
                            --> loaded only when actually called

[6] Immutable State         DeepImmutable + Zustand
                            --> predictable state changes

[7] Interruption            Save transcript BEFORE query loop
    Resilience              --> crash mid-API = resume from last message

[8] Dependency Injection    query() receives deps parameter
                            --> mockable for tests, swappable per mode
```

---

## 24. 전체 데이터 흐름 요약

```
End-to-End Data Flow
=====================

User runs "claude" in terminal
      |
      v
main.tsx
+-- Auth (OAuth / API key / Bedrock / Vertex)
+-- Model resolution (Opus for Max, Sonnet for others)
+-- Load settings + feature gates
+-- Collect context (Git status + CLAUDE.md)  [memoized]
+-- Build tool pool (built-in + MCP)
+-- Launch REPL  (or headless QueryEngine)
      |
      v
User types: "Fix the bug in auth.ts"
      |
      v
Normalize message for API
      |
      v
query() generator loop  <-----------------------------------------+
|                                                                  |
+-- [Pre-process] snip / microcompact / auto-compact if needed     |
+-- [API call] stream response from Claude API                     |
+-- [Error?] withhold & recover (413 -> compact, max_tok -> retry) |
+-- [Tool use?]                                                    |
|     YES --> permission check --> execute tool --> collect result  |
|             (pipeline: validate -> hooks -> rules -> classifier) -+
|     NO  --> display final response
|
+-- Record transcript to disk
+-- Track cost (per model: input/output/cache tokens)
+-- Await next user input
```

이 다이어그램이 Claude Code의 전체 동작을 하나의 그림으로 보여준다. 사용자가 터미널에서 Claude Code를 실행하면, `main.tsx`가 인증, 모델 해석, 설정 로딩, 컨텍스트 수집을 수행한 후 REPL을 시작한다. 사용자가 메시지를 입력하면, 메시지가 정규화되어 Claude API에 스트리밍으로 전송된다. 응답이 도착하면 텍스트는 즉시 화면에 표시되고, 도구 사용 요청은 권한 확인을 거쳐 실행된다. 도구 결과는 다시 API에 전달되고, 이 과정이 AI가 더 이상 도구를 사용하지 않을 때까지 반복된다. 대화가 길어지면 자동 압축이 작동하고, 에러가 발생하면 자동 복구가 시도된다. 모든 과정에서 비용이 추적되며, 세션이 끝나면 기록이 영속화된다.

---

> 출처: [wikidocs.net/338204](https://wikidocs.net/338204) — Claude Code 소스 코드 분석서
