# Release a new version

## Steps

1. **Update README version references** — Replace all `codescene-oss/pr-refactoring-agent@v<old>` with `@v<new>` in `README.md`.
2. **Verify `action.yml` defaults** — Confirm the `version` input still defaults to `'latest'` so the `main` branch always uses the latest agent binary.
3. **Commit** — Stage and commit with message: `docs: reference v<new> in examples` (or a more descriptive message if other changes are included).
4. **Tag** — Create tag `v<new>` on the new commit.
5. **Push** — Push both the commit and the tag: `git push origin main && git push origin v<new>`.

## Rules

- The `main` branch must always use `latest` as the default for the `version` input in `action.yml`. Never change this.
- README example workflows pin to a specific tag (e.g. `@v1.0.7`) so users get reproducible behavior.
- Tags follow the format `v<major>.<minor>.<patch>`.
- If other code changes are part of the release, commit those first or together with the version bump, then tag.
