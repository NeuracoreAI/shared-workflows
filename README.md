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
| `integration-ml-test.yaml` | `environment` (staging \| production), `test-path`, `extras`, `lfs` |

### Composite actions (`actions/`)

Referenced at the *step* level:

| Action | Purpose |
|---|---|
| `actions/setup-neuracore-daemon-from-source` | Checkout of the neuracore source (optional) + Rust toolchain + cargo cache + FFmpeg + build daemon binary + `pip install .` + verify the bundled daemon. Works both from consumer repos (frontend, with `path: neuracore`) and inside the `neuracore` repo itself (`checkout: "false"`, `path: "."`). Production-style installs from PyPI stay inline in the consumer workflows for now (they are a short pip install + verify, with no Rust/checkout machinery to share). |

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
- uses: NeuracoreAI/shared-workflows/actions/setup-neuracore-daemon-from-source@main
  with:
    path: neuracore
```

Notes:

- Python must be set up by the caller before the daemon setup actions (cache
  paths are repo-specific).

