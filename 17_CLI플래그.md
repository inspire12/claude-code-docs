# CLI 플래그

> 터미널에서 Claude Code를 실행할 때 전달할 수 있는 모든 옵션.

```bash
claude [플래그] [프롬프트]
```

## 핵심 플래그

**`-p, --print`**
Claude를 비대화형으로 실행. Claude가 프롬프트를 처리하고 응답을 출력한 후 종료합니다. REPL이 시작되지 않습니다.
```bash
claude -p "src/index.ts의 메인 함수를 설명해줘"
echo "이게 뭐야?" | claude -p
```
> ⚠️ `--print` 모드에서는 작업 공간 신뢰 다이얼로그가 건너뜁니다. 신뢰하는 디렉토리에서만 사용하세요.

**`--output-format <형식>`** (`--print`와 함께만 작동)
| 값 | 설명 |
|----|------|
| `text` | 일반 텍스트 출력 (기본값) |
| `json` | 완전한 결과가 담긴 단일 JSON 객체 |
| `stream-json` | 실시간 이벤트를 담은 줄바꿈으로 구분된 JSON 스트림 |

**`--verbose`**
상세 출력 활성화.

**`-v, --version`**
버전 번호 출력 후 종료.

## 세션 계속 플래그

**`-c, --continue`**
현재 디렉토리의 가장 최근 대화를 재개합니다.
```bash
claude --continue
claude -c "이제 테스트 추가해줘"
```

**`-r, --resume [session-id]`**
세션 ID로 대화를 재개합니다. 값 없이 사용하면 지난 세션 목록에서 선택합니다.
```bash
claude --resume
claude --resume 550e8400-e29b-41d4-a716-446655440000
claude --resume "auth refactor"
```

**`--fork-session`**
`--continue` 또는 `--resume`과 함께 사용해 재개된 대화에서 분기된 새 세션을 만듭니다.

**`-n, --name <이름>`**
세션 표시 이름 설정.

**`--no-session-persistence`**
세션 영속성 비활성화. `--print`와 함께만 작동합니다.

## 모델 및 기능 플래그

**`--model <모델>`**
세션 모델 설정. 별칭(`sonnet`, `opus`, `haiku`) 또는 전체 모델 ID(`claude-sonnet-4-6`) 허용.
```bash
claude --model sonnet
claude --model opus
claude --model claude-sonnet-4-6
```

**`--effort <레벨>`**
세션의 effort 레벨 설정: `low`, `medium` (기본), `high`, `max`.
```bash
claude --effort high "이 아키텍처를 검토해줘"
```

**`--fallback-model <모델>`**
기본 모델이 과부하될 때 자동 대체 모델 활성화. `--print`와 함께만 작동합니다.

## 권한 및 보안 플래그

**`--permission-mode <모드>`**
세션의 권한 모드 설정:
| 모드 | 동작 |
|------|------|
| `default` | 커맨드 실행 및 편집 전에 확인 요청 |
| `acceptEdits` | 파일 편집 자동 적용; 셸 커맨드는 여전히 확인 필요 |
| `plan` | 계획을 제안하고 실행 전 승인 대기 |
| `bypassPermissions` | 프롬프트 없이 모든 작업 실행 — 격리된 샌드박스 환경 전용 |

**`--dangerously-skip-permissions`**
모든 권한 검사를 우회합니다. `--permission-mode bypassPermissions`와 동일.
> ⚠️ 인터넷 접근 없는 샌드박스 환경에서만 사용하세요.

**`--allowed-tools <도구...>`** (별칭: `--allowedTools`)
Claude가 사용할 수 있는 도구의 쉼표 또는 공백 구분 목록.
```bash
claude --allowed-tools "Bash(git:*) Edit Read"
```

**`--disallowed-tools <도구...>`**
Claude가 사용할 수 없는 도구 목록.

**`--tools <도구...>`**
세션에 사용 가능한 정확한 내장 도구 집합 지정. `""`는 모든 도구 비활성화.

## 컨텍스트 및 프롬프트 플래그

**`--add-dir <디렉토리...>`**
도구 접근 컨텍스트에 하나 이상의 디렉토리 추가.
```bash
claude --add-dir /shared/libs --add-dir /shared/config
```

**`--system-prompt <프롬프트>`**
기본 시스템 프롬프트를 커스텀 프롬프트로 교체.

**`--append-system-prompt <텍스트>`**
기본 시스템 프롬프트에 텍스트 추가. `--system-prompt`와 달리 내장 지시사항을 유지합니다.

**`--mcp-config <설정...>`**
하나 이상의 JSON 설정 파일 또는 인라인 JSON 문자열에서 MCP 서버 로드.

**`--strict-mcp-config`**
`--mcp-config`의 MCP 서버만 사용하고 다른 모든 MCP 설정 무시.

**`--settings <파일-또는-json>`**
JSON 파일 경로 또는 인라인 JSON 문자열에서 추가 설정 로드.

**`--agents <json>`**
JSON 객체로 커스텀 에이전트를 인라인으로 정의.

## 출력 제어 플래그

**`--max-turns <n>`**
비대화형 모드의 에이전틱 턴 수 제한. `--print`와 함께만 작동합니다.

**`--max-budget-usd <금액>`**
API 호출 최대 지출 금액 설정. `--print`와 함께만 작동합니다.

**`--json-schema <스키마>`**
구조화된 출력 검증을 위한 JSON 스키마 제공.

## Worktree 플래그

**`-w, --worktree [이름]`**
세션에 새 git worktree 생성. PR 번호나 GitHub PR URL 허용.
```bash
claude --worktree
claude --worktree feature-auth
claude --worktree "#142"
```

**`--tmux`**
worktree와 함께 tmux 세션 생성. `--worktree` 필요.

## 디버그 플래그

**`-d, --debug [필터]`**
디버그 모드 활성화.
```bash
claude --debug
claude --debug "api,hooks"
claude --debug "!file,!1p"
```

**`--debug-file <경로>`**
디버그 로그를 인라인 표시 대신 파일에 저장.

**`--bare`**
최소 모드. 훅, LSP, 플러그인 동기화, 어트리뷰션, 자동 메모리, 백그라운드 프리페치, 키체인 읽기, `CLAUDE.md` 자동 발견을 모두 건너뜁니다. 스타트업 지연이 중요하고 이러한 기능이 필요하지 않은 스크립트 파이프라인에 사용하세요.

## 일반적인 플래그 조합

```bash
# JSON 출력과 함께 비대화형
claude -p "모든 내보낸 타입 목록" --output-format json

# CI에서 권한 우회 (샌드박스 환경 전용)
claude -p "전체 테스트 스위트 실행하고 실패 수정해줘" --dangerously-skip-permissions

# 마지막 세션 재개 후 비대화형으로 계속
claude --continue -p "이제 그것에 대한 테스트 작성해줘"

# 커스텀 MCP 설정으로 엄격한 격리
claude --mcp-config ./ci-mcp.json --strict-mcp-config -p "스키마 분석해줘"

# 시스템 프롬프트를 교체하지 않고 추가
claude --append-system-prompt "항상 JavaScript가 아닌 TypeScript를 출력하세요."
```
