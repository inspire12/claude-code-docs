# SDK 개요

> stdin/stdout 제어 프로토콜을 사용해 자체 도구에 Claude Code를 임베드하는 방법. SDK 세션 API, 메시지 타입, 출력 형식에 대한 레퍼런스.

Claude Code SDK는 다른 애플리케이션에 Claude Code를 임베드하기 위한 제어 프로토콜입니다 — IDE, 자동화 스크립트, CI/CD 파이프라인, 또는 서브프로세스를 스폰하고 stdin/stdout으로 통신할 수 있는 모든 호스트.

라이브러리 API를 직접 노출하는 대신, 구조화된 JSON 메시지 스트림을 통해 실행 중인 `claude` 프로세스와 통신합니다. 호스트 프로세스가 사용자 메시지와 제어 요청을 전송하면 CLI 프로세스가 어시스턴트 메시지, 도구 진행 이벤트, 결과 페이로드를 스트리밍합니다.

## 작동 방식

1. **Claude Code 프로세스 스폰** — `--output-format stream-json`과 `--print`로 시작 (비대화형 모드). stdin과 stdout을 호스트 프로세스로 파이핑.

```bash
claude --output-format stream-json --print --verbose
```

여러 프롬프트를 받는 영속 세션은 `--print`를 생략하고 세션 초기화 후 stdin에 `SDKUserMessage` 객체를 전송합니다.

2. **초기화 요청 전송** — stdin에 `control_request` (`subtype: "initialize"`) 작성. CLI가 `SDKControlInitializeResponse`로 응답합니다.

3. **stdout에서 메시지 스트리밍** — stdout에서 줄바꿈으로 구분된 JSON 읽기. 각 줄이 `SDKMessage` 유니온 타입 중 하나입니다.

4. **사용자 메시지 전송** — 대화를 계속하려면 stdin에 `SDKUserMessage` 객체 작성.

## 출력 형식

| 형식 | 설명 |
|------|------|
| `text` | 일반 텍스트 응답만. 대화형 모드의 기본값 |
| `json` | 완료 시 작성되는 단일 JSON 객체. 일회성 스크립트에 적합 |
| `stream-json` | 줄바꿈으로 구분된 JSON 스트림. 이벤트 발생 시 줄당 하나의 메시지. SDK 사용에 필수 |

## 제어 프로토콜 메시지

제어 프로토콜은 stdin/stdout에서 양방향으로 흐르는 두 가지 최상위 봉투 타입을 사용합니다.

### `SDKControlRequest` (호스트 → CLI)

```json
{
  "type": "control_request",
  "request_id": "<고유-문자열>",
  "request": { "subtype": "...", ...페이로드 }
}
```

### `SDKControlResponse` (CLI → 호스트)

```json
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "<에코된-id>",
    "response": { ...페이로드 }
  }
}
```

오류 시 `subtype`은 `"error"`이고 `error` 필드에 메시지가 포함됩니다.

## 초기화 요청

`initialize` 요청은 반드시 먼저 보내야 합니다. 세션을 설정하고 사용 가능한 기능을 반환합니다.

```json
{
  "type": "control_request",
  "request_id": "init-1",
  "request": {
    "subtype": "initialize",
    "systemPrompt": "당신은 CI 자동화 에이전트입니다.",
    "appendSystemPrompt": "항상 테스트 커버리지를 추가하세요.",
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hookCallbackIds": ["my-hook-id"]
        }
      ]
    },
    "agents": {
      "CodeReviewer": {
        "description": "코드 품질과 보안을 검토합니다.",
        "prompt": "당신은 전문 코드 리뷰어입니다...",
        "model": "opus"
      }
    }
  }
}
```

### 초기화 응답

```json
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "init-1",
    "response": {
      "commands": [...],
      "agents": [...],
      "output_style": "stream-json",
      "models": [...],
      "account": {
        "email": "user@example.com",
        "organization": "Acme Corp",
        "apiProvider": "firstParty"
      }
    }
  }
}
```

## 사용자 메시지

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "이 함수를 async/await으로 리팩터링해줘."
  },
  "parent_tool_use_id": null
}
```

- `parent_tool_use_id`: 이 메시지가 응답하는 도구 사용 ID, 최상위 사용자 메시지는 `null`
- `priority`: `'now'` | `'next'` | `'later'` — 비동기 메시지 큐잉 스케줄링 힌트

## SDK 메시지 스트림 타입

| 타입 | 설명 |
|------|------|
| `system` (`subtype: "init"`) | 세션 시작 시 한 번 출력. 활성 모델, 도구 목록, MCP 서버 상태, 권한 모드, 세션 ID 포함 |
| `assistant` | 모델이 턴을 생성할 때 출력. `tool_use` 블록 포함 가능 |
| `stream_event` | 스트리밍 중 부분 토큰 출력. 점진적 렌더링에 사용 |
| `tool_progress` | 몇 초 이상 걸리는 도구의 주기적 상태 업데이트 |
| `result` | 각 턴 종료 시 출력. `subtype`은 `"success"` 또는 오류 서브타입 |

**result 예시:**
```json
{
  "type": "result",
  "subtype": "success",
  "result": "함수가 리팩터링되었습니다.",
  "duration_ms": 4200,
  "total_cost_usd": 0.0042,
  "num_turns": 3,
  "is_error": false,
  "session_id": "abc123"
}
```

## 기타 제어 요청

| `subtype` | 방향 | 설명 |
|-----------|------|------|
| `interrupt` | 호스트 → CLI | 현재 턴 중단 |
| `set_permission_mode` | 호스트 → CLI | 활성 권한 모드 변경 |
| `set_model` | 호스트 → CLI | 세션 중간에 모델 전환 |
| `can_use_tool` | CLI → 호스트 | 도구 호출 권한 요청 |
| `mcp_status` | 호스트 → CLI | MCP 서버 연결 상태 가져오기 |
| `get_context_usage` | 호스트 → CLI | 컨텍스트 윈도우 사용량 가져오기 |
| `rewind_files` | 호스트 → CLI | 특정 메시지 이후 파일 변경 사항 되돌리기 |
| `hook_callback` | CLI → 호스트 | SDK에 등록된 훅 이벤트 전달 |

## 세션 관리 API

스크립팅 시나리오를 위해 SDK가 `~/.claude/`에 저장된 세션 트랜스크립트를 조작하는 함수를 내보냅니다:

```typescript
import {
  query,
  listSessions,
  getSessionInfo,
  getSessionMessages,
  forkSession,
  renameSession,
  tagSession,
} from '@anthropic-ai/claude-code'
```

**`query` — 프롬프트 실행 (주요 SDK 진입점):**
```typescript
for await (const message of query({
  prompt: '이 디렉토리에 어떤 파일이 있나요?',
  options: { cwd: '/my/project' }
})) {
  if (message.type === 'result') {
    console.log(message.result)
  }
}
```

**`listSessions` — 저장된 세션 목록:**
```typescript
const sessions = await listSessions({ dir: '/my/project', limit: 50 })
```

**`getSessionMessages` — 트랜스크립트 읽기:**
```typescript
const messages = await getSessionMessages(sessionId, {
  dir: '/my/project',
  includeSystemMessages: false,
})
```

**`forkSession` — 대화 분기:**
```typescript
const { sessionId: newId } = await forkSession(originalSessionId, {
  upToMessageId: 'msg-uuid',
  title: '실험적 분기',
})
```

## 사용 사례

**IDE 통합:** IDE가 영속 Claude Code 프로세스를 스폰하고 제어 프로토콜을 통해 메시지를 라우팅합니다. `PreToolUse` 훅 콜백으로 파일 편집을 가로채 적용 전에 IDE 네이티브 UI에 diff를 표시합니다.

**CI/CD 자동화:**
```bash
result=$(echo "diff를 검토하고 pass/fail을 출력해줘" | \
  claude --output-format json --print \
         --permission-mode bypassPermissions)
```

**헤드리스 에이전트:** TypeScript SDK에서 스트리밍 출력 형식으로 `query()`를 사용합니다. 지속 데몬 프로세스를 유지하면서 크론 스케줄로 작업을 실행합니다.
