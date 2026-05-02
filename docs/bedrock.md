# AWS Bedrock Support

This document covers both the setup guide for running Automaker with AWS Bedrock and the code changes made to support it.

---

## Setup Guide

### Prerequisites

- AWS account with Bedrock access enabled
- A Bedrock bearer token (`AWS_BEARER_TOKEN_BEDROCK`) and your AWS region

### 1. Configure environment variables

Create a `.env` file in the root of the repo (same directory as `docker-compose.yml`):

```bash
# Automaker web login key — use any string you want
AUTOMAKER_API_KEY=your-chosen-login-key

# AWS Bedrock authentication
AWS_BEARER_TOKEN_BEDROCK=your-bearer-token
AWS_REGION=us-east-1
```

### 2. Build and start containers

```bash
docker compose build
docker compose up -d
```

### 3. Log in to the UI

Open `http://localhost:3007` and enter the value you set for `AUTOMAKER_API_KEY`.

> If you didn't set `AUTOMAKER_API_KEY`, the server generates a random UUID on startup. Find it with:
>
> ```bash
> docker logs automaker-server 2>&1 | grep -A5 "API Key for Web Mode"
> ```

### 4. Add the AWS Bedrock provider

1. Go to **Settings → Claude → Custom Providers**
2. Click **Add Provider** and select the **AWS Bedrock** template
3. The form will pre-fill with 6 cross-region inference profile model IDs
4. No API key or base URL is needed — auth comes from the env vars set above
5. Click **Save**

### 5. Select a Bedrock model

In **Settings → Model Defaults**, choose any of the Bedrock models (e.g. `us.anthropic.claude-sonnet-4-6`) for the phases you want to run via Bedrock.

### Supported env vars (forwarded to Claude SDK subprocess)

| Variable                          | Purpose                                       |
| --------------------------------- | --------------------------------------------- |
| `AWS_BEARER_TOKEN_BEDROCK`        | Bearer token for Bedrock authentication       |
| `AWS_REGION`                      | AWS region (e.g. `us-east-1`)                 |
| `AWS_ACCESS_KEY_ID`               | IAM access key (alternative to bearer token)  |
| `AWS_SECRET_ACCESS_KEY`           | IAM secret key (alternative to bearer token)  |
| `AWS_SESSION_TOKEN`               | IAM session token (for temporary credentials) |
| `AWS_DEFAULT_REGION`              | Fallback region if `AWS_REGION` not set       |
| `AWS_PROFILE`                     | AWS named profile from `~/.aws/config`        |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable Claude Code auto memory feature       |

---

## Code Changes

### Overview

Bedrock support was added through the existing `ClaudeCompatibleProvider` system — no new provider class was needed. The key insight is that Bedrock uses `CLAUDE_CODE_USE_BEDROCK=1` plus AWS credentials instead of `ANTHROPIC_API_KEY` + a base URL.

### Files changed

#### `libs/types/src/settings.ts`

- Added `'bedrock'` to the `ClaudeCompatibleProviderType` union type
- Added a full Bedrock template to `CLAUDE_PROVIDER_TEMPLATES` with 6 cross-region inference profile model IDs:
  - `us.anthropic.claude-haiku-4-5-20251001`
  - `us.anthropic.claude-sonnet-4-6`
  - `us.anthropic.claude-opus-4-6`
  - `us.anthropic.claude-sonnet-4-5-20251015`
  - `us.anthropic.claude-opus-4-1`
  - `us.anthropic.claude-opus-4-5`
- Template sets `baseUrl: ''` and `defaultApiKeySource: 'env'` since Bedrock doesn't use an API key or custom endpoint

#### `apps/server/src/providers/claude-provider.ts`

- Added `AWS_ENV_VARS` array containing all AWS credential variables including `AWS_BEARER_TOKEN_BEDROCK`
- Restructured `buildEnv()` to branch on Bedrock vs non-Bedrock:
  - **Bedrock**: sets `CLAUDE_CODE_USE_BEDROCK=1`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`, forwards all AWS env vars and `CLAUDE_CODE_DISABLE_AUTO_MEMORY`
  - **Non-Bedrock**: existing behavior unchanged (sets `ANTHROPIC_API_KEY`/`ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`, model mappings, etc.)
- Fixed CLI OAuth auth: when no explicit API key is configured, inherit full `process.env` so macOS Keychain tokens are accessible to the Claude SDK subprocess
- Added Claude Opus 4.7 and Sonnet 4.7 as available models (Opus 4.7 set as default)

#### `apps/ui/src/components/views/settings-view/providers/claude-settings-tab/api-profiles-section.tsx`

- Added `bedrock: 'AWS Bedrock'` to `PROVIDER_TYPE_LABELS`
- Added `bedrock: 'bg-orange-500/20 text-orange-400'` to `PROVIDER_TYPE_COLORS`
- Added `bedrock` option to the Provider Type dropdown (for manual custom provider creation)
- Fixed form validation: Bedrock skips both API key and base URL requirements
- Replaced API key input with an info note showing required env vars when Bedrock is selected
- Hidden base URL and advanced options fields for Bedrock providers

#### `apps/ui/src/components/views/settings-view/model-defaults/phase-model-selector.tsx`

- Added `case 'bedrock': return AnthropicIcon` to all three provider icon switch statements so Bedrock models display the Anthropic icon in the model selector

#### `apps/server/tests/unit/providers/claude-provider.test.ts`

- Added two tests in `describe('buildEnv for Bedrock provider')`:
  1. Bedrock provider sets `CLAUDE_CODE_USE_BEDROCK=1`, passes AWS env vars, does not set `ANTHROPIC_API_KEY` or `ANTHROPIC_BASE_URL`
  2. Non-Bedrock provider does not set `CLAUDE_CODE_USE_BEDROCK`, sets `ANTHROPIC_BASE_URL` correctly
- Updated model count test from 5 to 7
- Added tests for Claude Opus 4.7 and Sonnet 4.7
- Updated default model assertion to Opus 4.7
