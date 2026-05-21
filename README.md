# CodeScene PR Refactoring Agent

> **Note**: This project is under active development and not yet ready for use.

A GitHub Action that brings automated code refactoring and code health improvements to your pull requests. Uses AI-powered analysis to guide refactoring based on CodeScene's code health metrics.

## Features

- 🔍 **Automatic code health analysis** - Identifies technical debt and code smells
- 🤖 **AI-guided refactoring** - Uses state-of-the-art LLMs to suggest and apply improvements
- 📊 **CodeScene integration** - Leverages CodeScene's battle-tested code health metrics
- 🔄 **PR-driven workflow** - Trigger refactorings directly from pull request comments
- 🎯 **Skill-based execution** - Pre-built refactoring skills for common scenarios

## Quick Start

### Basic Usage

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
      - uses: codescene-oss/pr-refactoring-agent@v1
        with:
          pr_number: ${{ github.event.issue.number }}
          command: ${{ github.event.comment.body }}
          push: true
          codescene_token: ${{ secrets.CODESCENE_ACCESS_TOKEN }}
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Then comment `/cs-agent` on any pull request to trigger the agent.

The action automatically:
- Fetches PR metadata
- Checks out the PR branch
- Configures git
- Runs the refactoring
- Pushes changes (if `push: true`)
- Posts a comment with the result

## Available Skills

The agent includes two pre-built refactoring skills:

- `skill:fix-code-health-degradations` - Fix only the Code Health regressions introduced by the PR, without touching pre-existing debt
- `skill:uplift-code-health` - Raise Code Health for selected files toward a target score, in measurable incremental steps

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `pr_number` | Pull request number (triggers PR checkout and comment) | No | - |
| `command` | The refactoring command to execute | Yes | - |
| `model` | AI model to use | No | `anthropic/claude-sonnet-4-20250514` |
| `version` | Version of the agent to use | No | `latest` |
| `create_branch` | Create a new branch before refactoring | No | - |
| `push` | Push changes to remote after refactoring | No | `false` |
| `remote` | Git remote name | No | `origin` |
| `github_token` | GitHub token for authentication | No | `${{ github.token }}` |
| `codescene_token` | CodeScene API access token | Yes | - |
| `anthropic_api_key` | Anthropic API key | No | - |
| `openai_api_key` | OpenAI API key | No | - |
| `google_api_key` | Google API key | No | - |
| `opencode_auth_json` | OpenCode auth JSON | No | - |

## Supported Models

The agent supports multiple AI providers:

- **Anthropic**: `anthropic/claude-sonnet-4-20250514`, `anthropic/claude-opus-4-20250514`
- **OpenAI**: `openai/gpt-4`, `openai/gpt-4-turbo`
- **Google**: `google/gemini-pro`
- **GitHub Copilot**: `github-copilot/gpt-4.1` (requires `opencode_auth_json`)

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
      - uses: codescene-oss/pr-refactoring-agent@v1
        with:
          pr_number: ${{ github.event.issue.number }}
          command: ${{ github.event.comment.body }}
          push: true
          codescene_token: ${{ secrets.CODESCENE_ACCESS_TOKEN }}
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```


## Required Secrets

Add these secrets to your repository (Settings → Secrets and variables → Actions):

### Required
- `CODESCENE_ACCESS_TOKEN` - Get from [CodeScene](https://codescene.com)

### At least one AI provider
- `ANTHROPIC_API_KEY` - Get from [Anthropic](https://console.anthropic.com)
- `OPENAI_API_KEY` - Get from [OpenAI](https://platform.openai.com)
- `GOOGLE_API_KEY` - Get from [Google AI Studio](https://aistudio.google.com)

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
- [Documentation](https://codescene.com/docs)
