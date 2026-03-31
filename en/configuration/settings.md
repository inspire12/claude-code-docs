> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Settings

> Configure Claude Code behavior through JSON settings files at the user, project, and managed levels.

Claude Code reads settings from JSON files at multiple scopes. Settings merge from lowest to highest priority, so more specific scopes override broader ones.

## Settings files

<Tabs>
  <Tab title="Global (user)">
    **Location:** `~/.claude/settings.json`

    Applies to every Claude Code session you run, across all projects. This is where you set personal preferences like your preferred model, theme, and cleanup policy.

    ```json  theme={null}
    {
      "model": "claude-opus-4-5",
      "cleanupPeriodDays": 30,
      "permissions": {
        "defaultMode": "acceptEdits"
      }
    }
    ```
  </Tab>

  <Tab title="Project (shared)">
    **Location:** `.claude/settings.json` in your project root

    Checked into source control. Use this for settings that should apply to everyone working on the project — permission rules, hook configurations, MCP servers, and environment variables.

    ```json  theme={null}
    {
      "permissions": {
        "allow": ["Bash(npm run *)", "Bash(git *)"],
        "deny": ["Bash(rm -rf *)"]
      },
      "hooks": {
        "PostToolUse": [
          {
            "matcher": "Write",
            "hooks": [{ "type": "command", "command": "npm run lint" }]
          }
        ]
      }
    }
    ```
  </Tab>

  <Tab title="Local (personal project)">
    **Location:** `.claude/settings.local.json` in your project root

    Not checked into source control (automatically added to `.gitignore`). Use this for personal overrides within a specific project — your own permission preferences or environment variables that shouldn't be shared.

    ```json  theme={null}
    {
      "permissions": {
        "defaultMode": "bypassPermissions"
      }
    }
    ```
  </Tab>

  <Tab title="Managed (enterprise)">
    **Location:** Platform-specific system path

    Set by administrators via MDM, registry (Windows), plist (macOS), or a managed settings file. Managed settings take highest priority and cannot be overridden by users or projects.

    See [Managed settings](#managed-settings-enterprise) below for details.
  </Tab>
</Tabs>

## Opening settings

Run `/config` inside any Claude Code session to open the settings UI. This opens the **Config** tab of the settings panel, where you can browse and edit the currently active settings for each scope.

You can also edit the JSON files directly. Claude Code reloads settings automatically when it detects file changes.

## Settings precedence

Settings merge from lowest to highest priority. Later sources override earlier ones for scalar values; arrays are concatenated and deduplicated.

```
Plugin defaults → User settings → Project settings → Local settings → Managed (policy) settings
```

<Note>
  Managed (policy) settings always take final precedence. An administrator setting `permissions.defaultMode` to `"default"` cannot be overridden by users or project files.
</Note>

## Settings reference

<AccordionGroup>
  <Accordion title="model">
    **Type:** `string` | **Scope:** Any

    Override the default model used by Claude Code. Accepts any model ID supported by your configured provider.

    ```json  theme={null}
    { "model": "claude-opus-4-5" }
    ```
  </Accordion>

  <Accordion title="permissions">
    **Type:** `object` | **Scope:** Any

    Controls which tools Claude can use and in what mode. See [Permissions](/concepts/permissions) for full documentation on rule syntax.

    | Field                          | Type        | Description                                                                               |
    | ------------------------------ | ----------- | ----------------------------------------------------------------------------------------- |
    | `allow`                        | `string[]`  | Rules for operations Claude may perform without asking                                    |
    | `deny`                         | `string[]`  | Rules for operations Claude is always blocked from                                        |
    | `ask`                          | `string[]`  | Rules for operations that always prompt for confirmation                                  |
    | `defaultMode`                  | `string`    | Default permission mode: `"default"`, `"acceptEdits"`, `"plan"`, or `"bypassPermissions"` |
    | `disableBypassPermissionsMode` | `"disable"` | Prevent users from entering bypass permissions mode                                       |
    | `additionalDirectories`        | `string[]`  | Extra directories Claude may access                                                       |

    ```json  theme={null}
    {
      "permissions": {
        "defaultMode": "acceptEdits",
        "allow": ["Bash(npm run *)", "Bash(git log *)"],
        "deny": ["Bash(curl *)"]
      }
    }
    ```
  </Accordion>

  <Accordion title="hooks">
    **Type:** `object` | **Scope:** Any

    Run custom shell commands before or after tool executions. See [Hooks](/guides/hooks) for full documentation.

    Supported hook events: `PreToolUse`, `PostToolUse`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `Stop`, `SubagentStop`, `PreCompact`, `PostCompact`.

    ```json  theme={null}
    {
      "hooks": {
        "PostToolUse": [
          {
            "matcher": "Write",
            "hooks": [{ "type": "command", "command": "prettier --write $CLAUDE_FILE_PATHS" }]
          }
        ]
      }
    }
    ```
  </Accordion>

  <Accordion title="cleanupPeriodDays">
    **Type:** `integer` (non-negative) | **Default:** `30` | **Scope:** Any

    Number of days to retain chat transcripts. Setting to `0` disables session persistence entirely — no transcripts are written and existing ones are deleted at startup.

    ```json  theme={null}
    { "cleanupPeriodDays": 7 }
    ```
  </Accordion>

  <Accordion title="env">
    **Type:** `object` | **Scope:** Any

    Environment variables to inject into every Claude Code session. Values are coerced to strings.

    ```json  theme={null}
    {
      "env": {
        "NODE_ENV": "development",
        "MY_API_URL": "https://api.example.com"
      }
    }
    ```
  </Accordion>

  <Accordion title="availableModels">
    **Type:** `string[]` | **Scope:** Managed

    Enterprise allowlist of models users can select. Accepts family aliases (`"opus"` allows any Opus version), version prefixes, or full model IDs. If `undefined`, all models are available. If an empty array, only the default model is available.

    ```json  theme={null}
    { "availableModels": ["claude-opus-4-5", "claude-sonnet-4"] }
    ```
  </Accordion>

  <Accordion title="allowedMcpServers / deniedMcpServers">
    **Type:** `object[]` | **Scope:** Any

    Enterprise allowlist and denylist for MCP servers. Each entry must have exactly one of `serverName`, `serverCommand`, or `serverUrl`. The denylist takes precedence — if a server matches both lists, it is blocked.

    ```json  theme={null}
    {
      "allowedMcpServers": [
        { "serverName": "my-db-server" },
        { "serverUrl": "https://mcp.example.com/*" }
      ],
      "deniedMcpServers": [
        { "serverCommand": ["npx", "malicious-server"] }
      ]
    }
    ```
  </Accordion>

  <Accordion title="worktree">
    **Type:** `object` | **Scope:** Any

    Configuration for `--worktree` flag behavior.

    | Field                | Type       | Description                                                                       |
    | -------------------- | ---------- | --------------------------------------------------------------------------------- |
    | `symlinkDirectories` | `string[]` | Directories to symlink from the main repo into worktrees (e.g., `"node_modules"`) |
    | `sparsePaths`        | `string[]` | Paths to check out in sparse mode, for faster worktrees in large monorepos        |

    ```json  theme={null}
    {
      "worktree": {
        "symlinkDirectories": ["node_modules", ".cache"],
        "sparsePaths": ["src/", "packages/my-service/"]
      }
    }
    ```
  </Accordion>

  <Accordion title="attribution">
    **Type:** `object` | **Scope:** Any

    Customize the attribution text Claude appends to commits and pull request descriptions.

    | Field    | Type     | Description                                                                      |
    | -------- | -------- | -------------------------------------------------------------------------------- |
    | `commit` | `string` | Attribution text (including any git trailers). Empty string removes attribution. |
    | `pr`     | `string` | Attribution text for PR descriptions. Empty string removes attribution.          |

    ```json  theme={null}
    {
      "attribution": {
        "commit": "Co-Authored-By: Claude <noreply@anthropic.com>",
        "pr": ""
      }
    }
    ```
  </Accordion>

  <Accordion title="language">
    **Type:** `string` | **Scope:** Any

    Preferred language for Claude responses and voice dictation. Accepts plain language names.

    ```json  theme={null}
    { "language": "japanese" }
    ```
  </Accordion>

  <Accordion title="sandbox">
    **Type:** `object` | **Scope:** Any

    Sandbox configuration for isolating Claude's tool executions. Contact your administrator for sandbox setup details.
  </Accordion>

  <Accordion title="alwaysThinkingEnabled">
    **Type:** `boolean` | **Default:** `true` | **Scope:** Any

    When `false`, extended thinking is disabled. When absent or `true`, thinking is enabled automatically for models that support it.

    ```json  theme={null}
    { "alwaysThinkingEnabled": false }
    ```
  </Accordion>

  <Accordion title="effortLevel">
    **Type:** `"low" | "medium" | "high"` | **Scope:** Any

    Persisted effort level for models that support adjustable thinking budgets.

    ```json  theme={null}
    { "effortLevel": "high" }
    ```
  </Accordion>

  <Accordion title="autoMemoryEnabled">
    **Type:** `boolean` | **Scope:** User, local

    Enable or disable auto-memory for this project. When `false`, Claude does not read from or write to the auto-memory directory.

    ```json  theme={null}
    { "autoMemoryEnabled": false }
    ```
  </Accordion>

  <Accordion title="autoMemoryDirectory">
    **Type:** `string` | **Scope:** User, local

    Custom path for auto-memory storage. Supports `~/` for home directory expansion. Ignored if set in project settings (`.claude/settings.json`) for security reasons. Defaults to `~/.claude/projects/<sanitized-cwd>/memory/`.

    ```json  theme={null}
    { "autoMemoryDirectory": "~/my-memory-store" }
    ```
  </Accordion>

  <Accordion title="claudeMdExcludes">
    **Type:** `string[]` | **Scope:** Any

    Glob patterns or absolute paths of `CLAUDE.md` files to exclude from loading. Patterns are matched against absolute paths using picomatch. Only applies to User, Project, and Local memory types — Managed files cannot be excluded.

    ```json  theme={null}
    {
      "claudeMdExcludes": [
        "/home/user/monorepo/vendor/CLAUDE.md",
        "**/third-party/.claude/rules/**"
      ]
    }
    ```
  </Accordion>

  <Accordion title="disableAllHooks">
    **Type:** `boolean` | **Scope:** Any

    Set to `true` to disable all hook and `statusLine` execution.

    ```json  theme={null}
    { "disableAllHooks": true }
    ```
  </Accordion>

  <Accordion title="respectGitignore">
    **Type:** `boolean` | **Default:** `true` | **Scope:** Any

    Whether the file picker respects `.gitignore` files. `.ignore` files are always respected regardless of this setting.

    ```json  theme={null}
    { "respectGitignore": false }
    ```
  </Accordion>

  <Accordion title="defaultShell">
    **Type:** `"bash" | "powershell"` | **Default:** `"bash"` | **Scope:** Any

    Default shell for `!` commands in the input box.

    ```json  theme={null}
    { "defaultShell": "powershell" }
    ```
  </Accordion>

  <Accordion title="forceLoginMethod">
    **Type:** `"claudeai" | "console"` | **Scope:** Managed

    Force a specific login method. Use `"claudeai"` for Claude Pro/Max accounts, or `"console"` for Anthropic Console billing.

    ```json  theme={null}
    { "forceLoginMethod": "console" }
    ```
  </Accordion>

  <Accordion title="apiKeyHelper">
    **Type:** `string` | **Scope:** User, managed

    Path to a script that outputs authentication values. The script is called to retrieve credentials dynamically instead of reading a static API key.

    ```json  theme={null}
    { "apiKeyHelper": "/usr/local/bin/get-claude-key.sh" }
    ```
  </Accordion>

  <Accordion title="syntaxHighlightingDisabled">
    **Type:** `boolean` | **Scope:** Any

    Disable syntax highlighting in diffs.

    ```json  theme={null}
    { "syntaxHighlightingDisabled": true }
    ```
  </Accordion>

  <Accordion title="prefersReducedMotion">
    **Type:** `boolean` | **Scope:** Any

    Reduce or disable animations (spinner shimmer, flash effects, etc.) for accessibility.

    ```json  theme={null}
    { "prefersReducedMotion": true }
    ```
  </Accordion>
</AccordionGroup>

## Managed settings (enterprise)

Administrators can push settings to all users via platform-native mechanisms. Managed settings take precedence over all user and project settings.

<Tabs>
  <Tab title="macOS">
    Deploy a plist file to `/Library/Preferences/` or via MDM (Jamf, Kandji, etc.) targeting `com.anthropic.claudecode`.
  </Tab>

  <Tab title="Windows">
    Write settings to the `HKLM\Software\Anthropic\Claude Code` registry key (machine-wide) or `HKCU\Software\Anthropic\Claude Code` (per-user, lower priority than HKLM).
  </Tab>

  <Tab title="File-based">
    Place a `managed-settings.json` file at the platform-specific managed path. For drop-in configuration fragments, create a `managed-settings.d/` directory alongside it. Files in that directory are sorted alphabetically and merged on top of the base file — later filenames win.

    ```
    managed-settings.json          # base settings (lowest precedence)
    managed-settings.d/
      10-security.json             # merged first
      20-model-policy.json         # merged second (wins over 10-)
    ```

    This convention matches systemd/sudoers drop-ins: independent teams can ship policy fragments without coordinating edits to a single file.
  </Tab>
</Tabs>

The following managed-only settings lock down user customization surfaces:

| Setting                           | Description                                                                                                                                                           |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `allowManagedHooksOnly`           | When `true`, only hooks defined in managed settings run. User, project, and local hooks are ignored.                                                                  |
| `allowManagedPermissionRulesOnly` | When `true`, only permission rules from managed settings are respected.                                                                                               |
| `allowManagedMcpServersOnly`      | When `true`, the `allowedMcpServers` allowlist is only read from managed settings.                                                                                    |
| `strictPluginOnlyCustomization`   | Lock specific customization surfaces (`"skills"`, `"agents"`, `"hooks"`, `"mcp"`) to plugin-only sources. Pass `true` to lock all four, or an array of surface names. |

<Warning>
  `allowManagedPermissionRulesOnly` ignores permission rules from user settings, project settings, local settings, and CLI arguments. Make sure your managed rules are comprehensive before enabling this.
</Warning>

## JSON schema

Add `$schema` to your settings file for editor autocompletion and validation:

```json  theme={null}
{
  "$schema": "https://schemas.anthropic.com/claude-code/settings.json"
}
```


Built with [Mintlify](https://mintlify.com).