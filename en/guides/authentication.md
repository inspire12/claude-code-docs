> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Authentication

> Configure how Claude Code authenticates with the Anthropic API, AWS Bedrock, or GCP Vertex AI.

Claude Code supports several authentication methods. The method you use depends on how you're accessing the API — directly through Anthropic, through a cloud provider, or via an API key helper script.

## Claude.ai OAuth (default)

When you run `claude` for the first time without any API key configured, Claude Code starts an OAuth flow using your claude.ai account.

<Steps>
  <Step title="Run Claude Code">
    Open your terminal and run:

    ```bash  theme={null}
    claude
    ```
  </Step>

  <Step title="Follow the login prompt">
    Claude Code will display a URL and prompt you to open it in your browser. Visit the URL, sign in to your claude.ai account, and grant authorization.
  </Step>

  <Step title="Return to the terminal">
    After authorizing in the browser, Claude Code receives the OAuth token automatically and stores it in secure storage (macOS Keychain on macOS, or a credentials file on other platforms). You're now authenticated.
  </Step>
</Steps>

<Note>
  OAuth tokens are refreshed automatically before they expire. You do not need to re-authenticate unless you explicitly log out or revoke access.
</Note>

## API key

You can authenticate using an Anthropic API key instead of OAuth.

<Tabs>
  <Tab title="Environment variable">
    Set the `ANTHROPIC_API_KEY` environment variable in your shell profile or before running `claude`:

    ```bash  theme={null}
    export ANTHROPIC_API_KEY=sk-ant-...
    ```

    When this variable is set, Claude Code uses it directly and does not prompt for OAuth.
  </Tab>

  <Tab title="Settings file">
    Add the `apiKeyHelper` setting to `~/.claude/settings.json` to run a shell command that outputs your API key. Claude Code executes this command and caches the result for 5 minutes (configurable with `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`):

    ```json  theme={null}
    {
      "apiKeyHelper": "cat ~/.anthropic/api-key"
    }
    ```

    The `apiKeyHelper` command must print only the API key to stdout and exit with code 0. Any stderr output is treated as an error.
  </Tab>
</Tabs>

<Warning>
  When `ANTHROPIC_API_KEY` is set or `apiKeyHelper` is configured, the OAuth flow is disabled. Claude Code will not attempt to use your claude.ai account.
</Warning>

## AWS Bedrock

To use Claude through Amazon Bedrock, set the `CLAUDE_CODE_USE_BEDROCK` environment variable and configure your AWS credentials.

<Steps>
  <Step title="Enable Bedrock mode">
    ```bash  theme={null}
    export CLAUDE_CODE_USE_BEDROCK=1
    ```
  </Step>

  <Step title="Configure AWS credentials">
    Claude Code uses the standard AWS credential chain. Any of the following work:

    * AWS credentials file (`~/.aws/credentials`)
    * Environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`
    * IAM roles (EC2 instance profiles, ECS task roles, etc.)
    * AWS SSO (`aws sso login`)
  </Step>

  <Step title="Set your region and model (optional)">
    ```bash  theme={null}
    export AWS_REGION=us-east-1
    ```

    Claude Code selects a Claude model automatically based on your Bedrock configuration.
  </Step>
</Steps>

### Automated AWS credential refresh

If your AWS session expires mid-session (for example, with short-lived SSO tokens), configure `awsAuthRefresh` in your settings to run a command that refreshes credentials automatically:

```json  theme={null}
{
  "awsAuthRefresh": "aws sso login --profile my-profile"
}
```

Claude Code runs this command when it detects that credentials have expired and streams the output so you can complete any browser-based flows.

To export credentials from a command (for example, `aws sts assume-role`), use `awsCredentialExport`:

```json  theme={null}
{
  "awsCredentialExport": "aws sts assume-role --role-arn arn:aws:iam::123456789012:role/MyRole --role-session-name claude-code --query Credentials --output json"
}
```

The command must output valid AWS STS JSON (with `Credentials.AccessKeyId`, `Credentials.SecretAccessKey`, and `Credentials.SessionToken`).

## GCP Vertex AI

To use Claude through Google Cloud Vertex AI, set the `CLAUDE_CODE_USE_VERTEX` environment variable and configure Application Default Credentials.

<Steps>
  <Step title="Enable Vertex mode">
    ```bash  theme={null}
    export CLAUDE_CODE_USE_VERTEX=1
    ```
  </Step>

  <Step title="Configure GCP credentials">
    Claude Code uses Google Application Default Credentials (ADC). Any of the following work:

    * `gcloud auth application-default login` (for interactive use)
    * Service account key file via `GOOGLE_APPLICATION_CREDENTIALS`
    * Workload Identity (for GKE)
  </Step>

  <Step title="Set your GCP project and region (optional)">
    ```bash  theme={null}
    export ANTHROPIC_VERTEX_PROJECT_ID=my-gcp-project
    export CLOUD_ML_REGION=us-central1
    ```
  </Step>
</Steps>

### Automated GCP credential refresh

Similar to Bedrock, configure `gcpAuthRefresh` to run a command when credentials expire:

```json  theme={null}
{
  "gcpAuthRefresh": "gcloud auth application-default login"
}
```

Claude Code checks whether your current GCP credentials are valid before running the command, so it only refreshes when necessary.

## Switching accounts

### Log in to a different account

Run the `/login` command from within Claude Code to start a new OAuth flow. This replaces any stored tokens with those for the new account:

```
/login
```

### Log out

Run the `/logout` command to remove stored credentials:

```
/logout
```

After logging out, Claude Code will prompt you to authenticate on the next run.

## Token expiry and refresh

OAuth tokens expire automatically. Claude Code handles refresh silently:

* Before each API request, Claude Code checks whether the access token is expired.
* If it is, Claude Code acquires a lock and refreshes using the stored refresh token.
* Multiple concurrent Claude Code instances coordinate via a lock file to avoid redundant refreshes.
* If a `401` response arrives from the API (for example, due to a clock skew between when the token was issued and the local check), Claude Code forces an immediate refresh without waiting for the local expiry time.

You do not need to do anything when tokens refresh. If refresh fails (for example, because you revoked access in the claude.ai settings), Claude Code will prompt you to run `/login`.

## Authentication priority

When multiple authentication sources are configured, Claude Code resolves them in this order:

1. `ANTHROPIC_AUTH_TOKEN` environment variable
2. `CLAUDE_CODE_OAUTH_TOKEN` environment variable
3. OAuth token from file descriptor (for managed deployments)
4. `apiKeyHelper` from settings
5. Stored claude.ai OAuth tokens (keychain or credentials file)
6. `ANTHROPIC_API_KEY` environment variable

<Tip>
  For CI and non-interactive environments, use `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`. These are checked before any interactive flows.
</Tip>


Built with [Mintlify](https://mintlify.com).