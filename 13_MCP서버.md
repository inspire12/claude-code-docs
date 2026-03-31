# MCP 서버

> Model Context Protocol 서버를 연결해 Claude Code의 기능을 데이터베이스, API, 커스텀 도구로 확장합니다.

MCP(Model Context Protocol)는 Claude Code가 외부 데이터 소스와 서비스에 연결할 수 있게 하는 오픈 표준입니다. MCP 서버를 추가하면 Claude가 새 도구에 접근할 수 있습니다 — 예를 들어 데이터베이스 쿼리, Jira 티켓 읽기, Slack 워크스페이스 상호작용 등.

## 서버 추가 방법

**CLI를 통해:**
```bash
# 기본 추가
claude mcp add <이름> -- <커맨드> [인수...]

# 예: filesystem MCP 서버 추가
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /tmp

# 범위 지정
claude mcp add --scope project filesystem -- npx -y @modelcontextprotocol/server-filesystem /tmp
claude mcp add --scope user my-db -- npx -y @my-org/mcp-server-postgres
```

**`--mcp-config` 플래그:**
```bash
claude --mcp-config ./my-mcp-config.json
```
CI 환경이나 설정 파일에 저장하지 않을 독립형 설정에 유용합니다.

## 설정 파일 형식

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
      "env": {
        "NODE_ENV": "production"
      }
    }
  }
}
```

**HTTP 원격 서버:**
```json
{
  "mcpServers": {
    "my-api": {
      "type": "http",
      "url": "https://mcp.example.com/v1",
      "headers": {
        "Authorization": "Bearer $MY_API_TOKEN"
      }
    }
  }
}
```

**SSE (Server-Sent Events):**
```json
{
  "mcpServers": {
    "events-server": {
      "type": "sse",
      "url": "https://mcp.example.com/sse"
    }
  }
}
```

`command`, `args`, `url`, `headers` 값은 `$VAR` 및 `${VAR}` 구문을 지원합니다. 참조 변수가 없으면 Claude Code가 경고를 로그하지만 연결을 시도합니다.

## 설정 범위

| 범위 | 위치 | 용도 |
|------|------|------|
| `project` | 현재 디렉토리의 `.mcp.json` | 팀 공유 서버 설정 |
| `user` | `~/.claude.json` (전역 설정) | 모든 곳에서 이용 가능한 개인 서버 |
| `local` | 현재 프로젝트의 `.claude/settings.local.json` | 개인 프로젝트별 재정의, VCS에 커밋되지 않음 |

같은 서버 이름이 여러 범위에 나타나면 `local` > `project` > `user` 순으로 우선합니다.

## 서버 관리

**활성화/비활성화:**
```
/mcp enable <server-name>
/mcp disable <server-name>
/mcp enable all
/mcp disable all
```

비활성화된 서버는 설정에 남아있지만 시작 시 연결되지 않습니다.

**서버 재연결:**
```
/mcp reconnect <server-name>
```

**서버 상태 확인:**
`/mcp`를 실행해 모든 설정된 서버와 현재 연결 상태를 확인합니다:
- **connected** — 서버가 실행 중이고 준비됨
- **pending** — 서버가 시작 중
- **failed** — 서버 연결 실패(오류 메시지 확인)
- **needs-auth** — OAuth 인증 필요
- **disabled** — 설정되었지만 꺼짐

## MCP 도구 호출 승인

Claude Code는 MCP 도구를 호출하기 전에 권한 프롬프트를 표시합니다. 도구 이름과 입력 인수를 보여줍니다:
- **한 번 허용** — 이 특정 호출 승인
- **항상 허용** — 이 세션에서 이 도구의 모든 호출 승인
- **거부** — 호출 차단; Claude가 오류를 받고 다른 접근을 시도

> 📝 자동 모드(`--allowedTools`)에서 MCP 도구는 허용 도구 목록에 전체 이름(`mcp__<server-name>__<tool-name>` 형식)을 포함시켜 사전 승인할 수 있습니다.

## 예시: filesystem 서버

```bash
# 1. 서버 추가
claude mcp add --scope project filesystem -- npx -y @modelcontextprotocol/server-filesystem /home/user/projects

# 2. 연결 확인
# /mcp 실행 → filesystem이 connected로 표시되는지 확인

# 3. 사용
# Claude가 이제 mcp__filesystem__read_file과 mcp__filesystem__write_file 도구로
# /home/user/projects의 파일을 읽고 쓸 수 있음
```

## 예시: 데이터베이스 서버

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "$DATABASE_URL"
      }
    }
  }
}
```

Claude Code 시작 전에 환경에서 `DATABASE_URL`을 설정하면 MCP 서버가 자동으로 받습니다.

## 공식 MCP 레지스트리

[modelcontextprotocol.io](https://modelcontextprotocol.io)에서 Anthropic과 커뮤니티가 관리하는 MCP 서버 레지스트리를 찾아볼 수 있습니다 — 데이터베이스, 생산성 도구, 클라우드 제공업체 등.

## 문제 해결

**서버가 'failed'로 표시됨:**
- 커맨드가 존재하고 실행 가능한지 확인: `which npx`
- 터미널에서 커맨드를 직접 실행해 오류 없이 시작되는지 확인
- 필요한 환경 변수(API 키 등)가 설정되었는지 확인
- `claude --debug`로 상세 연결 로그 확인

**MCP 도구가 나타나지 않음:**
연결되었지만 미인증 상태인 서버는 도구를 노출하지 않습니다. `/mcp`에서 **needs-auth** 상태를 확인하고 OAuth 흐름을 따르세요.

**Windows: npx 실패:**
```json
{
  "mcpServers": {
    "my-server": {
      "command": "cmd",
      "args": ["/c", "npx", "-y", "@my-org/mcp-server"]
    }
  }
}
```
