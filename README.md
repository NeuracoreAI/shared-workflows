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
| `integration-ml-test.yaml` | `environment` (staging \| production), `test-path`, `extras`, `lfs` |

### Composite actions (`actions/`)

Referenced at the *step* level:

| Action | Purpose |
|---|---|
| `actions/checkout-test-harness` | Checks out the calling repo's integration test suite at the right ref and prints a harness summary. staging: the caller's branch; production: the latest `v*` release tag, verified after checkout (an explicit `ref` input overrides both). Optional LFS pull. Exposes the resolved `ref` and the production `release-tag` (for installing the matching released package). |
| `actions/setup-neuracore-from-source` | Checkout of the neuracore source (optional) + Rust toolchain + cargo cache + FFmpeg + build daemon binary + `pip install .` + verify the bundled daemon. Works both from consumer repos (frontend, with `path: neuracore`) and inside the `neuracore` repo itself (`checkout: "false"`, `path: "."`). |
| `actions/setup-neuracore-from-pypi` | Installs the released `neuracore` wheel from PyPI, optionally pinned to a version (e.g. the `release-tag` output of `checkout-test-harness`), with optional pip extras and PyTorch CPU wheel index, then verifies the install by importing it. Used by production-style integration tests. |

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

