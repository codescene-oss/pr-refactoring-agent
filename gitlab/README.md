# GitLab CI Integration for Refactoring Agent

This directory contains GitLab CI/CD templates for running the refactoring agent on GitLab merge requests.

## Quick Start

### 1. Prerequisites

- GitLab v14.5+ (for `/run-pipeline` quick action support)
- Developer+ permissions in your GitLab project
- Access tokens for:
  - GitLab (api + write_repository scope)
  - CodeScene
  - Your AI model provider (Anthropic, OpenAI, etc.)

### 2. Configure CI/CD Variables

In your GitLab project, go to **Settings → CI/CD → Variables** and add:

| Variable | Description | Scope |
|----------|-------------|-------|
| `GITLAB_TOKEN` | Project access token with `api` and `write_repository` scopes | Required |
| `CS_ACCESS_TOKEN` | CodeScene API access token | Required |
| `ANTHROPIC_API_KEY` | Anthropic API key (or use `OPENAI_API_KEY`, etc.) | Required |
| `MODEL` | AI model to use (optional, default: `anthropic/claude-sonnet-4-20250514`) | Optional |

**To create a GitLab project access token:**
1. Go to **Settings → Access Tokens**
2. Create token with:
   - Name: `refactoring-agent`
   - Role: `Developer` or `Maintainer`
   - Scopes: `api`, `write_repository`

### 3. Include Template in `.gitlab-ci.yml`

Add this to your project's `.gitlab-ci.yml`:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/codescene-oss/pr-refactoring-agent/main/gitlab/refactoring-agent.yml'

# Optional: Override default settings
variables:
  MODEL: "anthropic/claude-sonnet-4-20250514"
```

Commit and push this file to your repository.

### 4. Use in Merge Requests

When you have a merge request open, comment with:

```
/run-pipeline COMMAND="skill:fix-code-health-degradations improve complexity"
```

The pipeline will:
1. Trigger automatically
2. Download the refactoring agent binary
3. Create an initial MR note: "🤖 Refactoring Agent ⏳ Running..."
4. Execute the refactoring skills
5. Commit and push changes
6. Update the MR note with results: "✅ Completed successfully"

## Usage Examples

### Single Skill

Fix code health degradations:
```
/run-pipeline COMMAND="skill:fix-code-health-degradations improve complexity"
```

Uplift code health:
```
/run-pipeline COMMAND="skill:uplift-code-health improve readability"
```

### Multiple Skills

Chain multiple skills together:
```
/run-pipeline COMMAND="skill:fix-code-health-degradations skill:uplift-code-health improve all"
```

### Custom Goals

Provide specific instructions:
```
/run-pipeline COMMAND="skill:fix-code-health-degradations refactor src/utils/ for better maintainability"
```

## Advanced Configuration

### Use Different Model

Override the model in your `.gitlab-ci.yml`:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/codescene-oss/pr-refactoring-agent/main/gitlab/refactoring-agent.yml'

variables:
  MODEL: "openai/gpt-4"
```

Or pass it per-pipeline:
```
/run-pipeline COMMAND="skill:fix-code-health-degradations improve complexity" MODEL="openai/gpt-4"
```

### Pin to Specific Agent Version

```yaml
variables:
  REFACTORING_AGENT_VERSION: "v0.1.5"  # default: latest
```

### Pre-configured Manual Jobs

If you prefer clicking buttons over typing commands, create predefined jobs:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/codescene-oss/pr-refactoring-agent/main/gitlab/refactoring-agent.yml'

# Complexity refactoring job
refactoring-agent:complexity:
  extends: .refactoring-agent-base
  when: manual
  allow_failure: true
  only:
    - merge_requests
  variables:
    COMMAND: "skill:fix-code-health-degradations improve complexity"

# Security refactoring job
refactoring-agent:security:
  extends: .refactoring-agent-base
  when: manual
  allow_failure: true
  only:
    - merge_requests
  variables:
    COMMAND: "skill:fix-code-health-degradations improve security"

# Readability refactoring job
refactoring-agent:readability:
  extends: .refactoring-agent-base
  when: manual
  allow_failure: true
  only:
    - merge_requests
  variables:
    COMMAND: "skill:uplift-code-health improve readability"
```

Now you can go to **CI/CD → Pipelines** and click the play button on the specific job you want.

## Self-Hosted GitLab

This template works with self-hosted GitLab instances (v14.5+). The `CI_API_V4_URL` environment variable is automatically set by GitLab CI and points to your instance's API endpoint.

No additional configuration needed!

## Troubleshooting

### "COMMAND variable not set"

**Cause**: You didn't use the correct `/run-pipeline` syntax.

**Solution**: Use:
```
/run-pipeline COMMAND="skill:fix-code-health-degradations ..."
```

Note the `COMMAND="..."` with quotes.

### Pipeline doesn't trigger

**Cause**: Insufficient permissions or GitLab version too old.

**Solution**:
- Verify you have Developer+ role in the project
- Check GitLab version: `/run-pipeline` requires v14.5+ (Nov 2021)
- Verify `.gitlab-ci.yml` includes the template

### "Failed to download binary from GitHub Releases"

**Cause**: Network access to github.com blocked or invalid version.

**Solution**:
- Verify GitLab runner has internet access to github.com
- Check if `REFACTORING_AGENT_VERSION` is set to a valid release
- Try pinning to a specific version: `REFACTORING_AGENT_VERSION: "v0.1.5"`

### "PRIVATE-TOKEN: 401 Unauthorized"

**Cause**: Invalid or missing GITLAB_TOKEN.

**Solution**:
- Verify `GITLAB_TOKEN` is set in **Settings → CI/CD → Variables**
- Check token has `api` and `write_repository` scopes
- Regenerate token if expired

### MR note not created

**Cause**: GITLAB_TOKEN doesn't have `api` scope or token is invalid.

**Solution**:
- Verify token has `api` scope (not just `write_repository`)
- Check token hasn't expired
- Note: Refactoring will continue even if note creation fails

### Changes not pushed

**Cause**: GITLAB_TOKEN doesn't have `write_repository` scope.

**Solution**:
- Add `write_repository` scope to token
- Or set `push: false` in command if you don't want automatic pushes

## Architecture

```
User comments on MR
  ↓
/run-pipeline COMMAND="..."
  ↓
GitLab triggers pipeline with COMMAND variable
  ↓
CI job downloads binary from GitHub Releases
  ↓
Prepends skill:gitlab-mr-comment if in MR context
  ↓
Runs refactoring-agent binary
  ↓
Binary creates initial MR note via GitLab API
  ↓
Binary executes refactoring skills
  ↓
Binary updates MR note with results
  ↓
Changes committed and pushed to MR branch
```

## Available Skills

- `skill:fix-code-health-degradations` - Fix code health issues
- `skill:uplift-code-health` - Improve code quality
- `skill:prioritizing-technical-debt` - Analyze and prioritize tech debt
- `skill:routing-work-with-code-ownership` - Find code owners
- And more...

See the [main README](../README.md) for complete skill documentation.

## Differences from GitHub Action

| Feature | GitHub Action | GitLab CI |
|---------|---------------|-----------|
| **Trigger** | Comment `/cs-agent skill:...` | Comment `/run-pipeline COMMAND="skill:..."` |
| **Syntax** | Natural comment | Must use `COMMAND="..."` variable |
| **Comment parsing** | Automatic | Variable-based |
| **Setup** | Composite action | Include remote template |
| **Permissions** | Auto via GITHUB_TOKEN | Manual project access token |

## Support

- **Issues**: https://github.com/codescene-oss/refactoring-agent/issues
- **Documentation**: https://codescene.io/docs/
- **Minimum GitLab version**: v14.5 (released Nov 22, 2021)

## License

Same as main project - see [LICENSE](../LICENSE)
