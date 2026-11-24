# github-workflows

Reusable GitHub Actions workflows for my projects.

| Workflow | What it does |
| --- | --- |
| [node-ci.yml](.github/workflows/node-ci.yml) | npm install → lint → typecheck → unit tests → build |
| [node-run.yml](.github/workflows/node-run.yml) | npm install → one command (e2e, generated-code checks, …), artifact upload on failure |
| [node-coverage.yml](.github/workflows/node-coverage.yml) | tests with coverage, markdown summary, sticky PR comment |
| [claude-issue-triage.yml](.github/workflows/claude-issue-triage.yml) | Claude labels new issues and posts a scoping comment |

## Versioning

Callers pin a major tag:

```yaml
uses: HappyKnuckles/github-workflows/.github/workflows/node-ci.yml@v1
```

To ship a change, publish a GitHub release tagged `vX.Y.Z`. The [release workflow](.github/workflows/release.yml)
then moves the major tag `vX` to that release, so every caller on `@vX` picks it up on its next run.

- Patch / minor (`v1.2.3` → `v1.3.0`): fixes and new optional inputs. Callers update automatically.
- Major (`v2.0.0`): renamed or removed inputs, changed defaults, anything a caller must adapt to. Callers move to `@v2` by hand.
- Prereleases (`v1.4.0-rc.1`) don't move the major tag; test them by pinning the exact tag.

```bash
gh release create v1.1.0 --generate-notes
```

## Usage

Reusable workflows run in the caller's context: `actions/checkout` checks out the calling repo, and
`github.event` is the caller's event. Permissions are capped by the calling job, so grant what the
workflow needs there. Check runs are named `<caller job> / <called job>` (e.g. `verify / verify`).

### node-ci

```yaml
jobs:
  verify:
    uses: HappyKnuckles/github-workflows/.github/workflows/node-ci.yml@v1
    with:
      npm-version: "11"
      typecheck-command: npx tsc -p tsconfig.spec.json --noEmit
      install-playwright: true
      test-command: npx ng test --configuration=ci
```

Inputs: `node-version` (`22`), `npm-version` (empty = bundled), `install-command` (`npm ci`),
`lint-command` (`npm run lint`), `typecheck-command` (empty), `install-playwright` (`false`),
`playwright-browsers` (`chromium`), `test-command` (`npm test`), `build-command` (`npm run build`),
`timeout-minutes` (`20`). An empty command skips its step.

### node-run

```yaml
jobs:
  e2e:
    uses: HappyKnuckles/github-workflows/.github/workflows/node-run.yml@v1
    with:
      command: npm run e2e
      install-playwright: true
      failure-artifact-name: playwright-report
      failure-artifact-path: |
        playwright-report
        test-results
```

Inputs: `command` (required), `node-version`, `npm-version`, `install-command`, `install-playwright`,
`playwright-browsers`, `failure-artifact-name`, `failure-artifact-path` (empty = no upload),
`failure-artifact-retention-days` (`7`), `timeout-minutes` (`20`).

### node-coverage

```yaml
jobs:
  coverage:
    permissions:
      contents: read
      pull-requests: write
    uses: HappyKnuckles/github-workflows/.github/workflows/node-coverage.yml@v1
    with:
      install-playwright: true
      summary-command: node scripts/coverage-summary.mjs
```

Inputs: `node-version`, `npm-version`, `install-command`, `install-playwright`, `playwright-browsers`,
`coverage-command` (`npm run test:coverage`), `summary-command` (prints markdown; empty = no summary or
comment), `comment-on-pr` (`true`), `timeout-minutes` (`20`).

### claude-issue-triage

```yaml
on:
  issues:
    types: [opened, labeled]
  issue_comment:
    types: [created]
  workflow_dispatch:
    inputs:
      issue_number:
        description: Issue number to (re-)triage
        type: number
        required: true

jobs:
  triage:
    permissions:
      contents: read
      issues: write
    uses: HappyKnuckles/github-workflows/.github/workflows/claude-issue-triage.yml@v1
    with:
      issue-number: ${{ github.event.issue.number || inputs.issue_number }}
      project-description: a short description of the project
    secrets:
      claude-code-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The job itself filters events: a new issue, the `trigger-label` being added, a comment containing
`mention` on an issue (not a PR), or a manual dispatch. `gh issue create` (splitting) is only allowed
on maintainer-triggered runs.

Inputs: `issue-number` (required), `project-description` (required), `domain-vocabulary`,
`example-file-link`, `where-it-lands-hints`, `watch-out-hints`, `label-rules`, `trigger-label`
(`triage`), `running-label` (`triaging`), `mention` (`@claude`), `model` (`claude-sonnet-5`),
`timeout-minutes` (`10`). Secret: `claude-code-oauth-token`.

The repo needs the `trigger-label` and `running-label` labels to exist.

## Access

A public repository can only call reusable workflows from a **public** repository, so this repo must be
public for public callers. Private callers additionally need *Settings → Actions → General → Access*
set to allow repositories owned by the user.
