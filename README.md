# ci-workflows

Shared GitHub Actions workflows for the repositories of ALERTua.

## Checks

pre-commit runs actionlint and the zizmor security audit on each workflow, together with the usual file checks. Install the git hook once, then each commit runs the checks:

```bash
pre-commit install
pre-commit run --all-files
```

## docker-build

`docker-build.yml` builds the Docker image of the caller repository and pushes it to `ghcr.io/<owner>/<repository>`. If buildkit on alert-server answers through the tailnet, the build runs there and uses the layer cache of alert-server. If not, the build runs on the GitHub-hosted runner and uses the GitHub Actions cache. The summary of the run shows `Builder: alert-server` or `Builder: github`.

The workflow joins the tailnet with the GitHub OIDC token through Tailscale workload identity federation. No repository needs a secret. Pull requests from forks get no OIDC token, so they always build on the GitHub-hosted runner. Runs of `pull_request_target` and `workflow_run` do not join the tailnet either.

Use it in a workflow. The triggers, `concurrency` and the tests stay in the caller, because a reusable workflow cannot have its own triggers:

```yaml
on:
  workflow_dispatch:
  push:
    branches: ["main"]
    tags: ["*"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    uses: ALERTua/ci-workflows/.github/workflows/docker-build.yml@v2
    permissions:
      contents: read
      packages: write
      id-token: write
```

Two stages of one Dockerfile, with a command before the build:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - target: cpu
            suffix: ""
          - target: cuda
            suffix: "-cuda"
    uses: ALERTua/ci-workflows/.github/workflows/docker-build.yml@v2
    permissions:
      contents: read
      packages: write
      id-token: write
    with:
      target: ${{ matrix.target }}
      suffix: ${{ matrix.suffix }}
      free-disk-space: ${{ matrix.target == 'cuda' }}
      prepare: git clone https://huggingface.co/spaces/<owner>/<space>
```

| Input | Default | Meaning |
|---|---|---|
| `target` | empty | Dockerfile stage to build. Empty builds the last stage. It also names the GitHub Actions cache scope. |
| `suffix` | empty | Suffix for each image tag, for example `-cuda` |
| `prepare` | empty | Shell commands that run after the checkout and before the build |
| `free-disk-space` | `false` | Remove big unused tools from the GitHub-hosted runner before a build on it |
| `latest-on-default-branch` | `true` | Tag each push to the default branch as `latest`. A release tag always gets `latest`. |
| `context` | `.` | Build context path |
| `alert-server` | `true` | Build on buildkit of alert-server when it answers. `false` always builds on the GitHub-hosted runner. |

The output `builder` is `alert-server` or `github`.

Tags of the image: the branch name, the git tag name, and for a semantic version tag `1.2.3`, `1.2` and `1`. A pull request run builds the image but does not push it.

These conditions must stay true:

- The Tailscale trust credential trusts the subject `repo:ALERTua/*` and gives `tag:github-ci`.
- The tailnet policy gives `tag:github-ci` only `tcp:1234` on alert-server.
- buildkit on alert-server listens on `100.76.242.86:1234`.

## pick-runner

`pick-runner.yml` chooses where a job runs. If the multirunner runner on alert-server is available, the job runs there. If not, the job runs on a GitHub-hosted runner. alert-server has no multirunner now, so `pick-runner` always gives the GitHub-hosted runner.

Use it in a workflow:

```yaml
jobs:
  pick-runner:
    uses: ALERTua/ci-workflows/.github/workflows/pick-runner.yml@v1
    permissions:
      id-token: write

  test:
    needs: pick-runner
    runs-on: ${{ fromJson(needs.pick-runner.outputs.runner) }}
    steps:
      - run: echo "runs on ${{ needs.pick-runner.outputs.runner }}"
```

When a reusable workflow calls `pick-runner`, its caller must also give `id-token: write`.

| Input | Default | Meaning |
|---|---|---|
| `self-hosted` | `["self-hosted","alert-server"]` | JSON `runs-on` value for the alert-server runner |
| `fallback` | `"ubuntu-latest"` | JSON `runs-on` value when alert-server has no runner |

The output `runner` is a JSON value. Pass it through `fromJson`.
