# shared-workflows

Centralized GitHub Actions reusable workflows and composite actions for Neuracore repositories.

_Note: This is a public repository so no workflows that are only shared among private repos should be included._

## Contents

### Reusable workflows (`.github/workflows/`)

Called with `uses:` at the *job* level from a consumer repo:

| Workflow | Inputs |
|---|---|
| `pr-check-commit-messages.yaml` | `valid-prefixes`, `runs-on` |
| `pr-check-label.yaml` | `required-labels`, `runs-on` |
| `pr-changelog-reminder.yaml`| `trigger-labels`, `changelog-path`, `runs-on` |
| `code-freeze-gate.yaml` | `runs-on`; secret `actions-read-token` (actions:read on the integration-test repos) |
| `integration-ml-test.yaml` | `environment` (staging \| production), `test-path`, `extras`, `lfs`, `data-daemon-diagnostics` |

### Composite actions (`actions/`)

Referenced at the *step* level:

| Action | Purpose |
|---|---|
| `actions/checkout-test-harness` | Checks out the calling repo's integration test suite at the right ref and prints a harness summary. staging: the caller's branch; production: the latest `v*` release tag, verified after checkout (an explicit `ref` input overrides both). Optional LFS pull. Exposes the resolved `ref` and the production `release-tag` (for installing the matching released package). |
| `actions/setup-neuracore-from-source` | Checkout of the neuracore source (optional) + Rust toolchain + cargo cache + FFmpeg + build daemon binary + `pip install -e .` + verify the bundled daemon. Works both from consumer repos (frontend, with `path: neuracore`) and inside the `neuracore` repo itself (`checkout: "false"`, `path: "."`). Exposes the installed `neuracore-version` and `neuracore-types-version`, plus the `neuracore-types-sha` the git ref resolved to. |
| `actions/setup-neuracore-from-pypi` | Installs the released `neuracore` wheel from PyPI, optionally pinned to a version (e.g. the `release-tag` output of `checkout-test-harness`), with optional pip extras and PyTorch CPU wheel index, then verifies the install by importing it. Used by production-style integration tests. Exposes the installed `neuracore-version` and `neuracore-types-version`. |
| `actions/publish-ci-manifest` | Publishes a JSON manifest of what a run built or deployed -- the ref, the resolved package versions, the environment -- as a `ci-manifest-*` run artifact. Best-effort: every step is `continue-on-error`, so it can never fail the deploy or test it describes. See "CI dashboard manifests" below. |
| `actions/upload-data-daemon-diagnostics` | Best-effort upload of daemon logs and SQLite state, with optional explicit analytics paths. Includes hidden files, retains artifacts for 14 days by default, and links the artifact from the job summary. Matrix callers supply their distinguishing axes in `artifact-name`; run and rerun context is appended automatically. |

## Consumer usage

Reusable workflow (job level):

```yaml
jobs:
  check-format:
    uses: NeuracoreAI/shared-workflows/.github/workflows/pr-check-commit-messages.yaml@main
    permissions:
      pull-requests: write
      contents: read
```

Composite action (step level):

```yaml
- uses: NeuracoreAI/shared-workflows/actions/setup-neuracore-from-source@main
  with:
    path: neuracore
```

Notes:

- Python must be set up by the caller before the neuracore setup actions (cache
  paths are repo-specific).

## CI dashboard manifests

The [CI dashboard](https://ci-dashboard-neuracore.pages.dev) used to recover deployed package
versions by parsing `pip`/`npm` output out of job logs, which broke every time a repo changed
build tooling. Instead, a run states what it built and the dashboard reads it back:

```yaml
- uses: NeuracoreAI/shared-workflows/actions/publish-ci-manifest@main
  if: always()
  with:
    kind: deploy              # or integration-test
    service: backend
    environment: production
    built-ref: ${{ inputs.release_tag }}   # the ref CHECKED OUT, not github.ref
    built-sha: ${{ steps.checkout.outputs.sha }}
    packages: |
      neuracore-types=11.10.0
    extra: '{"neuracore_training_target": "v13.5.0"}'
```

The contract the dashboard relies on:

- The artifact name always starts with `ci-manifest`, and the file inside is `manifest.json`.
  `artifact-label` distinguishes several manifests within one run; the prefix is not
  configurable.
- `built-ref` / `built-sha` are what was *checked out*. A deploy that takes a release tag as a
  workflow input must pass it — `github.sha` is the tip of the branch the workflow was
  dispatched from, which is a different commit, and the dashboard would otherwise read the
  wrong `uv.lock` / `package.json` / `.env`.
- `packages` values must be resolved versions, not ranges. Where the install happens inside a
  container, read them back out of the built image rather than off the runner.
- Artifacts are retained for 90 days by default; older runs fall back to the dashboard's
  legacy log parsing.
