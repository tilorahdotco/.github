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
| `node-ci.yml` | Jobs `Install`, then parallel `Lint`, `Typecheck`, `Test`, then `Build`. Empty input skips that job. | `node-version`, `package-manager`, `working-directory`, `install`, `lint`, `typecheck`, `test`, `build`, `runner` | none |
| `links.yml` | Link check with lychee. | `args`, `runner` | none |
| `changes.yml` | Job `Changes`. Output `code` is `true` unless only ignored files changed. | `ignore`, `runner` | none |
| `security.yml` | Parallel jobs `Secret scan` (gitleaks CLI) and `Dependency audit`. | `secret-scan`, `audit`, `audit-level`, `working-directory`, `runner` | none |
| `deploy-cpanel.yml` | Job `Deploy` (cPanel Git pull), then `Verify live` (HTTP 200 and byte compare). | `trigger`, `cpanel-repo`, `branch`, `host` (required), `verify-url`, `live-base-url`, `live-files`, `runner` | `CPANEL_USER`, `CPANEL_PASSWORD` |
| `deploy-cloudflare.yml` | Job `Build`, then `Deploy` (wrangler) from the built artifact. Manual release only. | `node-version`, `install`, `build`, `check`, `wrangler-version`, `command`, `runner` | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` |

Rules:
- Pin every `uses:` to a full commit SHA. Dependabot keeps the pins in this repository current.
- Callers own triggers, concurrency and job names.
- Public repositories run on GitHub-hosted runners only.
- Keep secrets at the organization level, limited to the repositories that use them.

## Workflow shape

Every repository workflow should read as a graph of small named jobs joined by
`needs:`, like a release pipeline. The Actions run page then shows stages, fan-out
and fan-in instead of one long job.

Stages, left to right:

1. `Changes`: decide whether code changed (docs-only skip, below).
2. Parallel checks: `CI` group (`Install`, `Lint`, `Typecheck`, `Test`, `Build`) and `Security` group (`Secret scan`, `Dependency audit`). A caller matrix adds more groups.
3. `CI gate`: one job that fans in every check and is the only required check.
4. Deploy group: `Deploy`, then `Verify live`.
5. `Announce`: last job, reports the result.

Rules:
- Job ids are short and lowercase (`ci`, `security`, `deploy`). Job `name:` values are plain human text ("Node CI", "CI gate").
- A job that calls a reusable workflow renders as a grouped box. Its `name:` is the box title and the called jobs are the rows inside.
- Use `needs:` for every real dependency. Do not chain jobs that could run in parallel.
- Use a `strategy.matrix` on a caller job for fan-out (packages, Node versions, shards). Put the matrix value in `name:` so each box has its own title.
- An empty command input skips its job. Skipped jobs show as grey nodes and do not fail the graph.
- Keep the job count reasonable. Each job on a GitHub-hosted runner bills at least one minute and repeats checkout and setup. `node-ci.yml` shares dependencies through the cache, not artifacts.
- Required checks change names with the graph. After a repository adopts a new pin, set its required check to `CI gate` only.

### Docs-only skip

Callers skip CI and deploy workflows for documentation changes with `paths-ignore`.
Never put `paths-ignore` on the security workflow: a docs-only change can still contain a secret,
so the secret scan runs on every change.

```yaml
on:
  pull_request:
    paths-ignore: ["**/*.md", "docs/**", ".github/ISSUE_TEMPLATE/**", ".github/pull_request_template.md"]
  push:
    branches: [main]
    paths-ignore: ["**/*.md", "docs/**", ".github/ISSUE_TEMPLATE/**", ".github/pull_request_template.md"]
```

Docs-only commits can also carry `[skip ci]` in the message. Both ways leave required checks
waiting forever, because the workflow never starts or its check never reports.

### Required checks: always report

For a repository with required checks, do not use `paths-ignore` or `[skip ci]`. Run the workflow every time
and let a `Changes` job decide. Skipped jobs then report as skipped, and the `CI gate` job reports success:

- The caller needs only `permissions: contents: read`.
- `Changes` gates CI and deploy. It never gates `Security`, which runs on every change.
- `changes.yml` globs are anchored at the repository root like `paths-ignore`: `*` stays inside one directory, `**/` matches any depth, and `docs/**` does not match `src/docs/x.js`.
- Do not require the nested check names (`ci / Lint`). When the caller job is skipped, those checks never exist. Require `CI gate` only.
- `CI gate` uses `if: always()`, so it runs even when every check was skipped. It fails when any needed job failed or was cancelled.

### Example caller

Copy to `.github/workflows/ci.yml`. Replace `<sha>` with the full commit SHA of the shared workflows.
The graph is `Changes` then `CI` and `Security` in parallel, then `CI gate`, then `Deploy`, then `Announce`.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  changes:
    name: Changes
    uses: tilorahdotco/.github/.github/workflows/changes.yml@<sha>

  ci:
    name: CI ${{ matrix.package }}
    needs: [changes]
    if: needs.changes.outputs.code == 'true'
    strategy:
      fail-fast: false
      matrix:
        package: [web, api]
    uses: tilorahdotco/.github/.github/workflows/node-ci.yml@<sha>
    with:
      working-directory: packages/${{ matrix.package }}
      lint: npm run lint
      typecheck: npm run typecheck
      test: npm test
      build: npm run build

  security:
    name: Security
    uses: tilorahdotco/.github/.github/workflows/security.yml@<sha>

  gate:
    name: CI gate
    needs: [changes, ci, security]
    if: always()
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - name: Check results
        env:
          RESULTS: ${{ join(needs.*.result, ' ') }}
        run: |
          echo "needed job results: $RESULTS"
          case " $RESULTS " in *" failure "*|*" cancelled "*) exit 1 ;; esac

  deploy:
    name: Deploy site
    needs: [changes, gate]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main' && needs.gate.result == 'success' && needs.changes.outputs.code == 'true'
    uses: tilorahdotco/.github/.github/workflows/deploy-cpanel.yml@<sha>
    with:
      host: example.com
      cpanel-repo: example
      verify-url: https://example.com/
    secrets:
      CPANEL_USER: ${{ secrets.CPANEL_USER }}
      CPANEL_PASSWORD: ${{ secrets.CPANEL_PASSWORD }}

  announce:
    name: Announce
    needs: [gate, deploy]
    if: always() && needs.gate.result != 'skipped'
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - name: Write run summary
        env:
          GATE: ${{ needs.gate.result }}
          DEPLOY: ${{ needs.deploy.result }}
        run: echo "CI gate $GATE, deploy $DEPLOY, commit $GITHUB_SHA" >> "$GITHUB_STEP_SUMMARY"
```

For a single package, drop the matrix and the `working-directory` line. The caller job is then one box named `CI`.
When a caller job skips (docs-only), `gate` still passes and only `Changes`, `CI gate` and `Announce` run. `Deploy site` skips too, because `code` is not `true`.
