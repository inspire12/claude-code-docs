> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Environment variables

> Environment variables that Claude Code reads to configure authentication, API access, behavior, and runtime options.

Claude Code reads environment variables at startup. These variables let you configure authentication, point to custom API endpoints, tune runtime behavior, and control which features are active — without modifying settings files.

## Authentication

<ParamField path="ANTHROPIC_API_KEY" type="string">
  API key for authenticating directly with the Anthropic API. When set, Claude Code uses this key instead of (or in addition to) OAuth credentials. Takes priority over OAuth when present in most contexts.

  ```bash  theme={null}
  export ANTHROPIC_API_KEY="sk-ant-..."
  ```
</ParamField>

<ParamField path="ANTHROPIC_AUTH_TOKEN" type="string">
  Alternative authentication token. Used in contexts where `ANTHROPIC_API_KEY` is not applicable.
</ParamField>

<ParamField path="ANTHROPIC_BASE_URL" type="string">
  Override the Anthropic API base URL. Useful for pointing Claude Code at a proxy, staging environment, or compatible third-party endpoint.

  ```bash  theme={null}
  export ANTHROPIC_BASE_URL="https://my-proxy.example.com"
  ```
</ParamField>

<ParamField path="CLAUDE_CODE_API_BASE_URL" type="string">
  Claude Code-specific API base URL override. Takes precedence over `ANTHROPIC_BASE_URL` when set.
</ParamField>

<ParamField path="ANTHROPIC_BEDROCK_BASE_URL" type="string">
  Base URL for AWS Bedrock API access. Set this when routing Claude Code through a Bedrock endpoint.
</ParamField>

<ParamField path="ANTHROPIC_VERTEX_PROJECT_ID" type="string">
  Google Cloud project ID for Vertex AI access. Required when using Claude Code through Google Cloud's Vertex AI platform.
</ParamField>

<ParamField path="CLAUDE_CODE_USE_BEDROCK" type="string">
  Set to `1` or `true` to use AWS Bedrock as the API provider.
</ParamField>

<ParamField path="CLAUDE_CODE_USE_FOUNDRY" type="string">
  Set to `1` or `true` to use Anthropic Foundry as the API provider.
</ParamField>

<ParamField path="CLAUDE_CODE_OAUTH_TOKEN" type="string">
  OAuth access token to use for authentication directly, bypassing the interactive login flow. Useful for automated environments.
</ParamField>

## Configuration paths

<ParamField path="CLAUDE_CONFIG_DIR" type="string" default="~/.claude">
  Override the directory where Claude Code stores its configuration, settings, and transcripts. Defaults to `~/.claude` in your home directory.

  ```bash  theme={null}
  export CLAUDE_CONFIG_DIR="/opt/claude-config"
  ```
</ParamField>

<ParamField path="CLAUDE_CODE_MANAGED_SETTINGS_PATH" type="string">
  Override the path to the managed settings file. Useful in enterprise environments where the default platform path is not appropriate.
</ParamField>

## Model selection

<ParamField path="ANTHROPIC_MODEL" type="string">
  Default model to use. Overridden by the `model` setting in settings files and by explicit `--model` flags.
</ParamField>

<ParamField path="CLAUDE_CODE_SUBAGENT_MODEL" type="string">
  Model to use for sub-agent tasks spawned by the main agent. When not set, sub-agents use the same model as the main session.
</ParamField>

<ParamField path="CLAUDE_CODE_AUTO_MODE_MODEL" type="string">
  Model to use when running in auto mode. Defaults to the main session model when not specified.
</ParamField>

## Behavior toggles

<ParamField path="CLAUDE_CODE_REMOTE" type="string">
  Set to `1` or `true` to enable remote/container mode. In this mode, Claude Code adjusts its behavior for non-interactive environments — extended API timeouts (120s vs. 300s), suppressed interactive prompts, and adapted output formatting.

  ```bash  theme={null}
  export CLAUDE_CODE_REMOTE=1
  ```
</ParamField>

<ParamField path="CLAUDE_CODE_SIMPLE" type="string">
  Set to `1` or `true` (or pass `--bare`) to run in bare mode. Bare mode skips hooks, LSP integration, plugin sync, skill directory walks, attribution, background prefetches, and all keychain/credential reads. Authentication must be provided via `ANTHROPIC_API_KEY` or `apiKeyHelper` in `--settings`. Useful for lightweight scripted use.
</ParamField>

<ParamField path="DISABLE_AUTO_COMPACT" type="string">
  Set to `1` or `true` to disable automatic context compaction. When set, Claude Code will not compact the conversation context even when it approaches the model's context limit.

  ```bash  theme={null}
  export DISABLE_AUTO_COMPACT=1
  ```
</ParamField>

<ParamField path="CLAUDE_CODE_DISABLE_BACKGROUND_TASKS" type="boolean">
  Set to `1` or `true` to disable background task execution. When enabled, the `run_in_background` parameter is removed from Bash and PowerShell tools, and Claude cannot run shell commands as background processes.
</ParamField>

<ParamField path="CLAUDE_CODE_DISABLE_THINKING" type="string">
  Set to `1` or `true` to disable extended thinking for all API calls, regardless of model support.
</ParamField>

<ParamField path="CLAUDE_CODE_DISABLE_AUTO_MEMORY" type="string">
  Set to `1` or `true` to disable automatic memory. When disabled, Claude does not read from or write to the auto-memory directory. Set to `0` or `false` to explicitly enable it (useful when other conditions would disable it, such as bare mode).

  Auto-memory is also disabled automatically in bare mode (`CLAUDE_CODE_SIMPLE`) and remote mode (`CLAUDE_CODE_REMOTE`) unless explicitly enabled.
</ParamField>

<ParamField path="CLAUDE_CODE_DISABLE_CLAUDE_MDS" type="string">
  Set to `1` or `true` to completely disable loading of all `CLAUDE.md` memory files. Claude will not read any CLAUDE.md, CLAUDE.local.md, or `.claude/rules/*.md` files.
</ParamField>

<ParamField path="CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC" type="string">
  Set to `1` or `true` to suppress analytics, telemetry, and other non-essential network requests.
</ParamField>

<ParamField path="CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD" type="string">
  Set to `1` or `true` to load `CLAUDE.md` files from directories added via `--add-dir`. By default, additional directories do not have their CLAUDE.md files loaded.
</ParamField>

<ParamField path="CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR" type="string">
  Set to `1` or `true` to reset the working directory back to the original project root after each Bash command. Prevents one command from changing the CWD for subsequent commands.
</ParamField>

## Resource limits

<ParamField path="CLAUDE_CODE_MAX_OUTPUT_TOKENS" type="number">
  Override the maximum number of output tokens per API response. When not set, Claude Code uses the model's default maximum. Useful for controlling costs in automated workflows.

  ```bash  theme={null}
  export CLAUDE_CODE_MAX_OUTPUT_TOKENS=4096
  ```
</ParamField>

<ParamField path="CLAUDE_CODE_MAX_CONTEXT_TOKENS" type="number">
  Override the maximum context window size. When set, Claude Code uses this value instead of the model's reported context limit.
</ParamField>

<ParamField path="BASH_MAX_OUTPUT_LENGTH" type="number">
  Maximum number of characters captured from Bash command output. Output exceeding this limit is truncated. Useful for preventing very large command outputs from consuming context.

  ```bash  theme={null}
  export BASH_MAX_OUTPUT_LENGTH=50000
  ```
</ParamField>

<ParamField path="API_TIMEOUT_MS" type="number">
  Override the API request timeout in milliseconds. Defaults to 300,000ms (5 minutes) for standard mode and 120,000ms (2 minutes) in remote mode.

  ```bash  theme={null}
  export API_TIMEOUT_MS=60000
  ```
</ParamField>

## Telemetry and observability

<ParamField path="CLAUDE_CODE_ENABLE_TELEMETRY" type="string">
  Set to `1` or `true` to enable OpenTelemetry export of traces, metrics, and logs. Requires additional OTEL configuration (endpoint, headers, etc.) to be set via standard OpenTelemetry environment variables.

  ```bash  theme={null}
  export CLAUDE_CODE_ENABLE_TELEMETRY=1
  export OTEL_EXPORTER_OTLP_ENDPOINT="https://otel.example.com"
  ```
</ParamField>

<ParamField path="CLAUDE_CODE_JSONL_TRANSCRIPT" type="string">
  Path to a file where Claude Code writes a JSONL transcript of the session. Each line is a JSON object representing a conversation event.

  ```bash  theme={null}
  export CLAUDE_CODE_JSONL_TRANSCRIPT="/tmp/session.jsonl"
  ```
</ParamField>

## Node.js runtime

<ParamField path="NODE_OPTIONS" type="string">
  Standard Node.js options passed to the runtime. Claude Code reads this to detect flags like `--max-old-space-size` and adjusts its behavior accordingly. Claude Code itself sets this internally to configure heap size for large sessions.

  ```bash  theme={null}
  export NODE_OPTIONS="--max-old-space-size=4096"
  ```

  <Warning>
    Avoid setting `NODE_OPTIONS` to values that include code execution flags. Claude Code validates shell environments and treats some `NODE_OPTIONS` values as requiring confirmation before running Bash commands.
  </Warning>
</ParamField>

## Host platform override

<ParamField path="CLAUDE_CODE_HOST_PLATFORM" type="string">
  Override the reported host platform for analytics. Accepts `"win32"`, `"darwin"`, or `"linux"`. Useful when Claude Code runs in a container (where `process.platform` reports the container OS) but the actual host platform differs.

  ```bash  theme={null}
  export CLAUDE_CODE_HOST_PLATFORM=darwin
  ```
</ParamField>

## Cloud provider region overrides

Claude Code supports per-model Vertex AI region overrides. Set the corresponding environment variable to route a specific model to a different region.

| Model prefix        | Environment variable              |
| ------------------- | --------------------------------- |
| `claude-haiku-4-5`  | `VERTEX_REGION_CLAUDE_HAIKU_4_5`  |
| `claude-3-5-haiku`  | `VERTEX_REGION_CLAUDE_3_5_HAIKU`  |
| `claude-3-5-sonnet` | `VERTEX_REGION_CLAUDE_3_5_SONNET` |
| `claude-3-7-sonnet` | `VERTEX_REGION_CLAUDE_3_7_SONNET` |
| `claude-opus-4-1`   | `VERTEX_REGION_CLAUDE_4_1_OPUS`   |
| `claude-opus-4`     | `VERTEX_REGION_CLAUDE_4_0_OPUS`   |
| `claude-sonnet-4-6` | `VERTEX_REGION_CLAUDE_4_6_SONNET` |
| `claude-sonnet-4-5` | `VERTEX_REGION_CLAUDE_4_5_SONNET` |
| `claude-sonnet-4`   | `VERTEX_REGION_CLAUDE_4_0_SONNET` |

```bash  theme={null}
# Route Opus 4 to a specific Vertex region
export VERTEX_REGION_CLAUDE_4_0_OPUS="us-central1"
```

The default Vertex region is controlled by `CLOUD_ML_REGION` (defaults to `us-east5`).

## AWS credentials

For Bedrock access, Claude Code respects standard AWS credential environment variables:

<ParamField path="AWS_REGION" type="string">
  AWS region for Bedrock API calls. Falls back to `AWS_DEFAULT_REGION`, then defaults to `us-east-1`.
</ParamField>

<ParamField path="AWS_DEFAULT_REGION" type="string">
  Fallback AWS region when `AWS_REGION` is not set.
</ParamField>

## Setting environment variables for all sessions

You can set environment variables that apply to every Claude Code session using the `env` field in your settings file, rather than setting them in your shell profile:

```json  theme={null}
{
  "env": {
    "DISABLE_AUTO_COMPACT": "1",
    "BASH_MAX_OUTPUT_LENGTH": "30000"
  }
}
```

See [Settings](/configuration/settings) for documentation on the `env` field.


Built with [Mintlify](https://mintlify.com).