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
| `links.yml` | Link check with lychee. | `args`, `root-dir`, `runner` | none |
| `changes.yml` | Job `Changes`. Output `code` is `true` unless only ignored files changed. | `ignore`, `runner` | none |
| `security.yml` | Parallel jobs `Secret scan` (gitleaks CLI: the PR commits on a pull request, else the history of the checked-out commit, never other branches) and `Dependency audit`. | `secret-scan`, `audit`, `audit-level`, `working-directory`, `runner` | none |
| `deploy-cpanel.yml` | Job `Deploy` (cPanel Git pull), then `Verify live` (HTTP 200 and byte compare). | `trigger`, `cpanel-repo`, `branch`, `host` (required), `verify-url`, `live-base-url`, `live-files`, `runner` | `CPANEL_USER`, `CPANEL_PASSWORD` |
| `deploy-cloudflare.yml` | Job `Build`, then `Deploy` (wrangler) from the built artifact. Manual release only. | `node-version`, `install`, `build`, `check`, `wrangler-version`, `command`, `runner` | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` |
| `pr-agent.yml` | Job `PR-Agent`: AI review of each PR with PR-Agent on Claude (Opus 5.5 reviews, Haiku 5.5 for `/ask` and `/improve`), and `/review`, `/improve`, `/ask` comments. No stored key: it exchanges GitHub's identity token for a 15-minute Anthropic token. | `runner`, `timeout-minutes` | none |
| `closure-comment.yml` | Job `Closure needs a comment`: reopens an issue, or an unmerged PR, closed without a closer comment of 10 or more characters. Skips bots, closes by a merged PR or commit, and a tracker sync within 5 minutes after a merged PR. Caller triggers: `issues: [closed]` and `pull_request: [closed]` (`pull_request_target: [closed]` in public repositories, so fork PRs get a write token), with `issues: write`, `pull-requests: write`, `contents: read`. | `runner` (default `vars.LIGHT_RUNNER`, then `cjbb2-ci`), `contributing-link`, `timeout-minutes` | none |

Rules:
- Pin every `uses:` to a full commit SHA. Dependabot keeps the pins in this repository current. The one exception is `pr-agent.yml`, which callers use at `@main` (see PR-Agent review).
- Callers own triggers, concurrency and job names.
- Public repositories run on GitHub-hosted runners only: pass `runner: ubuntu-latest`.
- Default runners: `node-ci.yml`, `links.yml`, `changes.yml`, `security.yml` and `closure-comment.yml` default to the free self-hosted `cjbb2-ci` runner (no Docker, no sudo). `pr-agent.yml` (Docker, no stored secrets) defaults to `'["self-hosted", "cjbb-docker"]'` (ephemeral Docker-capable self-hosted runner on cjbb, private repositories only; public repositories must pass `runner: ubuntu-latest`). `deploy-cpanel.yml` and `deploy-cloudflare.yml` (secrets, deploy) default to GitHub-hosted `ubuntu-latest`. A job that reads a secret other than `GITHUB_TOKEN`, runs `sudo` or apt, or deploys stays on `ubuntu-latest`.
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
    runs-on: cjbb2-ci
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
    runs-on: cjbb2-ci
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

### Link check for static sites

Root-relative links (`href="/about/"`) resolve against `root-dir`, a path relative to the repository root. Leave it empty for repositories without root-relative links.

```yaml
jobs:
  links:
    uses: tilorahdotco/.github/.github/workflows/links.yml@<sha>
    with:
      root-dir: .
      args: --verbose --no-progress --exclude-mail '**/*.html'
```

## PR-Agent review

Copy this caller to `.github/workflows/pr-agent.yml` in the repository. The file
name and the `@main` reference both matter. The Anthropic federation rule
`pr-agent-org` gives a token only to this workflow at `refs/heads/main` of this
repository, called from a `tilorahdotco` repository's
`.github/workflows/pr-agent.yml`, on a `pull_request` or `issue_comment` event.
This is the one exception to the SHA pin rule above: a copy of the workflow on
another branch, commit or fork gets no token, and a change merged to `main` here
reaches every repository at once. Only organization owners can push to `main`.

```yaml
name: PR-Agent review
on:
  pull_request:
    types: [opened, reopened, ready_for_review]
  issue_comment:
    types: [created]
permissions:
  contents: read
  pull-requests: write
  issues: write
  id-token: write # GitHub's identity token, exchanged for a short-lived Anthropic token
concurrency:
  # One review at a time per PR. Every comment gets its own group, so a command is
  # never replaced by a later comment or event; a command can run beside a review.
  group: pr-agent-${{ github.event.pull_request.number || github.event.issue.number }}-${{ github.event.comment.id || 'review' }}
  cancel-in-progress: false
jobs:
  review:
    name: PR-Agent review
    # Only PR events and the three commands start a run. The shared workflow checks
    # again: drafts, fork PRs, the author's write access and `--config.` arguments.
    if: github.event_name == 'pull_request' || (github.event.issue.pull_request && (startsWith(github.event.comment.body, '/review') || startsWith(github.event.comment.body, '/ask') || startsWith(github.event.comment.body, '/improve')))
    uses: tilorahdotco/.github/.github/workflows/pr-agent.yml@main
    # Uses the default ["self-hosted", "cjbb-docker"] runner (private repos only).
    # Public repositories must pass `with: { runner: ubuntu-latest }`.
```

- A repository that requires SHA pins (intersects) cannot use `@main`. It pins the
  commit of this workflow instead, and that exact commit must be listed in the
  rule's condition (Claude Console, Workload identity, rule `pr-agent-org`).
- Tokens act as the `pr-agents` service account and work only in the `pr-cicd`
  Anthropic workspace, which has its own spend limit. No repository needs an
  Anthropic secret.
- Reviews (automatic, or a `/review` comment) use Claude Opus 5.5 at high effort.
  The `/ask` and `/improve` comment commands use Claude Haiku 5.5 at medium effort.
  Sonnet 5.5 runs only when the chosen model fails. The workflow's settings win
  over a `model` in a repository's `.pr_agent.toml`; use that file for
  `[pr_reviewer] extra_instructions`. No other PR-Agent command runs.
- A fork PR gets no automatic review. A maintainer's `/review` comment on it runs
  from the default branch and reviews it; PR-Agent reads the diff and runs no PR code.
- The PR-Agent image is pinned by digest in the org action `actions/pr-agent`, which the workflow pins by commit. Use a runner with Docker:
  the default `["self-hosted", "cjbb-docker"]` runner for private repositories, or GitHub-hosted (`ubuntu-latest`) for public repositories, not the `cjbb2-ci` runner, which has no Docker.
- A failed exchange fails the job with Anthropic's error type. The deny reason is
  in the Claude Console under Workload identity, History.
