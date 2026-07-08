# CodeScene PR Refactoring Agent

A GitHub Action that enables CodeScene's PR Refactoring Agent so reviewers can trigger Code Health-guided refactoring directly from pull requests.

It keeps refactoring inside the normal review flow while giving teams a consistent way to improve maintainability and prevent regressions.

## Features

- 🔍 **Automatic code health analysis** - Identifies technical debt and code smells
- 🤖 **AI-guided refactoring** - Uses state-of-the-art LLMs to suggest and apply improvements
- 📊 **CodeScene integration** - Leverages CodeScene's battle-tested code health metrics
- 🔄 **PR-driven workflow** - Trigger refactorings directly from pull request comments
- 🎯 **Skill-based execution** - Pre-built refactoring skills for common scenarios

## Quick Start

1. Configure your repository secrets for CodeScene and at least one supported AI provider.
2. Add the workflow below to your repository.
3. Run the refactoring agent by clicking the fix button in the pull request (requires CodeScene PR integration to be enabled).

## Required: Configure repository secrets

Add these secrets to your repository (Settings → Secrets and variables → Actions):

- `CODESCENE_ACCESS_TOKEN` - Get it using the option that matches your setup:
  - CodeScene Cloud PAT: create it at [Create a Personal Access Token](https://codescene.io/users/me/pat).
  - CodeScene on-prem PAT: log in to your CodeScene instance, open `Configuration`, go to `Authentication`, then create a token under `Personal Access Tokens`. You can also go directly to `https://<your-cs-host><:port>/configuration/user/token`. You must also pass `codescene_onprem_url` in the workflow (e.g. `codescene_onprem_url: https://codescene.mycompany.com`).

**At least one AI provider:**
- `ANTHROPIC_API_KEY` - Get from [Anthropic](https://console.anthropic.com)
- `OPENAI_API_KEY` - Get from [OpenAI](https://platform.openai.com)
- `GOOGLE_API_KEY` - Get from [Google AI Studio](https://aistudio.google.com)
- `OPENCODE_AUTH_JSON` - OpenCode auth JSON, used to authenticate GitHub Copilot models. See [Using GitHub Copilot models](#using-github-copilot-models).

![Example of the pull request fix button flow](./assets/images/pr-fix-button-example.png)

_Example of how the pull request fix button looks when the refactoring agent is available._

### Required: add the agent workflow to GitHub

Add this workflow to your repository at `.github/workflows/refactoring-agent.yml`:

```yaml
name: PR Refactoring Agent

on:
  issue_comment:
    types: [created]

jobs:
  refactor:
    # Only run on PR comments that start with /cs-agent
    if: github.event.issue.pull_request != null && contains(github.event.comment.body, '/cs-agent')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
    steps:
      - uses: codescene-oss/pr-refactoring-agent@v1.0.8
        with:
          pr_number: ${{ github.event.issue.number }}
          command: ${{ github.event.comment.body }}
          model: 'anthropic/claude-sonnet-4-6-20251101'
          codescene_token: ${{ secrets.CODESCENE_ACCESS_TOKEN }}
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Then comment `/cs-agent` on any pull request to trigger the agent.

The action automatically:
- Fetches PR metadata
- Checks out the PR branch
- Configures git
- Runs the refactoring
- Pushes changes to the PR branch
- Posts a comment with the result

## 💡 The quality of the agent depends on the model

The refactoring quality of the agent depends heavily on the strength of the backing LLM, so use one of the strongest available models from a supported provider.

The refactoring agent supports models from Anthropic, OpenAI, and Google directly via API keys, as well as GitHub Copilot models (see below).

## Using GitHub Copilot models

You can point the agent at GitHub Copilot models (e.g. `github-copilot/claude-sonnet-4.5`) by passing an OpenCode auth JSON to the action via the `opencode_auth_json` input.

> [!IMPORTANT]
> GitHub Copilot only supports the interactive **OAuth device flow** for third-party tools like OpenCode. There is **no officially supported API key or Personal Access Token (PAT) flow** for headless/CI use. The setup below works by generating OAuth tokens locally (once, interactively) and reusing them in CI. See [Caveats](#caveats) below.

### 1. Generate the auth JSON locally

Authenticate OpenCode with GitHub Copilot on your own machine using the OAuth device flow:

```bash
opencode auth login
```

Select **GitHub Copilot** and follow the prompt to authorize the device at [github.com/login/device](https://github.com/login/device).

OpenCode stores the resulting credentials in its auth file at `~/.local/share/opencode/auth.json`. The GitHub Copilot entry looks like this:

```json
{"github-copilot":{"type":"oauth","access":"...","refresh":"...","expires":0}}
```

### 2. Add the auth JSON as a repository secret

Copy the `github-copilot` object from your `auth.json` (it must be valid single-line JSON) and add it as a repository secret named `OPENCODE_AUTH_JSON` (Settings → Secrets and variables → Actions).

The value should be the full JSON object, for example:

```json
{"github-copilot":{"type":"oauth","access":"...","refresh":"...","expires":0}}
```

### 3. Reference it in your workflow

Pass the secret via `opencode_auth_json` and set `model` to a `github-copilot/*` model:

```yaml
name: PR Refactoring Agent

on:
  issue_comment:
    types: [created]

jobs:
  refactor:
    if: github.event.issue.pull_request != null && contains(github.event.comment.body, '/cs-agent')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
    steps:
      - uses: codescene-oss/pr-refactoring-agent@v1.0.8
        with:
          pr_number: ${{ github.event.issue.number }}
          command: ${{ github.event.comment.body }}
          model: 'github-copilot/claude-sonnet-4.5'
          codescene_token: ${{ secrets.CODESCENE_ACCESS_TOKEN }}
          opencode_auth_json: ${{ secrets.OPENCODE_AUTH_JSON }}
```

### Caveats

- **No official headless support.** GitHub does not officially support connecting OpenCode (or other third-party tools) to GitHub Copilot without the interactive OAuth flow. The approach above is a supported-in-practice workaround: you complete the OAuth flow locally and reuse the tokens in CI.
- **Tokens can expire.** OAuth `access` tokens are short-lived; the `refresh` token is used to obtain new ones. If the agent stops authenticating, re-run `opencode auth login` locally and update the `OPENCODE_AUTH_JSON` secret.
- **Tread carefully with the PAT + `Copilot-Integration-Id` header workaround.** A commonly shared alternative injects a Personal Access Token and sets the `Copilot-Integration-Id: copilot-developer-cli` header. It can work, but GitHub has stated this is **not supported for third-party tools and that it could get accounts flagged for abuse** — so weigh that risk before relying on it. Background and GitHub's official response are tracked in [anomalyco/opencode#12258](https://github.com/anomalyco/opencode/issues/12258).

## Available Skills

The agent includes two pre-built refactoring skills:

- `skill:fix-code-health-degradations` - Fix only the Code Health regressions introduced by the PR, without touching pre-existing debt
- `skill:uplift-code-health` - Raise Code Health for selected files toward a target score, in measurable incremental steps

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `command` | The refactoring command to execute | Yes | - |
| `codescene_token` | CodeScene API access token | Yes | - |
| `model` | AI model to use | Yes | - |
| `pr_number` | Pull request number (triggers PR checkout and comment) | No | - |
| `version` | Version of the agent to use | No | `latest` |
| `github_token` | GitHub token for authentication | No | `${{ github.token }}` |
| `codescene_onprem_url` | CodeScene on-premises URL | No | - |
| `anthropic_api_key` | Anthropic API key | No | - |
| `openai_api_key` | OpenAI API key | No | - |
| `google_api_key` | Google API key | No | - |
| `opencode_auth_json` | OpenCode auth JSON (required for `github-copilot/*` models). See [Using GitHub Copilot models](#using-github-copilot-models) | No | - |
| `ca_bundle` | Path to a custom CA certificate bundle (PEM format) for SSL/TLS verification | No | - |

## Example Workflows

### Trigger on PR Comment

```yaml
name: PR Refactoring

on:
  issue_comment:
    types: [created]

jobs:
  refactor:
    if: github.event.issue.pull_request != null && contains(github.event.comment.body, '/cs-agent')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
    steps:
      - uses: codescene-oss/pr-refactoring-agent@v1.0.8
        with:
          pr_number: ${{ github.event.issue.number }}
          command: ${{ github.event.comment.body }}
          model: 'anthropic/claude-sonnet-4-6-20251101'
          codescene_token: ${{ secrets.CODESCENE_ACCESS_TOKEN }}
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```


## Platform Support

The action supports Linux runners (amd64, aarch64).

## How It Works

1. **Download**: The action downloads the appropriate pre-built binary for your platform
2. **Analyze**: CodeScene analyzes your code for health issues and technical debt
3. **Refactor**: The AI model generates and applies improvements based on CodeScene's guidance
4. **Commit**: Changes are automatically committed (and optionally pushed) to your branch

## License

Copyright CodeScene AB. See [LICENSE](LICENSE) for terms.

## Support

- [GitHub Issues](https://github.com/codescene-oss/pr-refactoring-agent/issues)
- [Documentation](https://codescene.io/docs/developer-tools/pr-refactoring-agent.html)
