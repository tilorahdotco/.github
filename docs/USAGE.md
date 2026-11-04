# Shared workflows

Reusable GitHub Actions workflows for Tilorah repositories. Call them from a
repository workflow with `uses:` and pin the commit SHA.

```yaml
jobs:
  ci:
    uses: tilorahdotco/.github/.github/workflows/node-ci.yml@<commit-sha>
    with:
      install: npm ci
      test: npm test
```

| Workflow | What it does | Main inputs | Secrets |
| --- | --- | --- | --- |
| `node-ci.yml` | Install, lint, typecheck, test, build. Empty input skips a step. | `node-version`, `package-manager`, `working-directory`, `install`, `lint`, `typecheck`, `test`, `build`, `runner` | none |
| `links.yml` | Link check with lychee. | `args`, `runner` | none |
| `security.yml` | Secret scan (gitleaks CLI) and dependency audit. | `secret-scan`, `audit`, `audit-level`, `working-directory`, `runner` | none |
| `deploy-cpanel.yml` | cPanel Git pull and deploy, then live checks. | `trigger`, `cpanel-repo`, `branch`, `host` (required), `verify-url`, `live-base-url`, `live-files`, `runner` | `CPANEL_USER`, `CPANEL_PASSWORD` |
| `deploy-cloudflare.yml` | Cloudflare Workers deploy with wrangler. Manual release only. | `node-version`, `install`, `build`, `check`, `wrangler-version`, `command`, `runner` | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` |

Rules:
- Pin every `uses:` to a full commit SHA. Dependabot keeps the pins in this repository current.
- Callers own triggers, concurrency and job names.
- Public repositories run on GitHub-hosted runners only.
- Keep secrets at the organization level, limited to the repositories that use them.
