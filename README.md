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

The workflow joins the tailnet with the GitHub OIDC token through Tailscale workload identity federation. No repository needs a secret. Pull requests from forks get no OIDC token, so they always build on the GitHub-hosted runner. Pull requests of Dependabot, and runs of `pull_request_target` and `workflow_run`, do not join the tailnet either.

### Image tags

| Event | Push | Tags |
|---|---|---|
| push to the default branch | yes | `edge` |
| push to another branch that the triggers name, for example `dev` | yes | the branch name |
| git tag `v1.2.3` or `1.2.3` | yes | `1.2.3`, `1.2`, `1`, and `latest` when it is the highest stable version of the repository |
| git tag `v0.3.1` | yes | `0.3.1`, `0.3`, and `latest` as above, but no `0` |
| pre-release `v1.3.0-rc.1` | yes | `1.3.0-rc.1` |
| pull request | no | none |
| manual run on the default branch or on a git tag | yes | as the push of the same ref |
| manual run on another branch | only with `push: true` | the branch name |

`latest` means the newest release, never the newest commit. A fix of an older line, for example `v1.0.5` after `v2.0.0`, moves `1` and `1.0` but not `latest`. With `suffix`, each tag gets the suffix, for example `edge-cuda` and `latest-cuda`.

Use it in a workflow. The triggers, `concurrency` and the tests stay in the caller, because a reusable workflow cannot have its own triggers. Do not ignore `.github/` in the `pull_request` trigger, so that the pull requests of Dependabot for the actions also run:

```yaml
on:
  workflow_dispatch:
  push:
    branches: ["main"]
    tags: ["*"]
  pull_request:
    branches: ["main"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    uses: ALERTua/ci-workflows/.github/workflows/docker-build.yml@v4.0.0
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
    uses: ALERTua/ci-workflows/.github/workflows/docker-build.yml@v4.0.0
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
| `context` | `.` | Build context path |
| `alert-server` | `true` | Build on buildkit of alert-server when it answers. `false` always builds on the GitHub-hosted runner. |
| `push` | `false` | Push the image of a manual run on a branch other than the default branch, with the branch name as the tag |

The output `builder` is `alert-server` or `github`. Each build also keeps its result as the artifact `build-result-<target>` for one day.

## pr-report

`pr-report.yml` comments one summary of all `docker-build.yml` builds of the run on the open pull request of the branch. A tool that follows the pull request then learns that all builds ended. Call it after the build jobs:

```yaml
jobs:
  build:
    # the build job or the matrix of build jobs, as above
  report:
    needs: build
    if: always()
    uses: ALERTua/ci-workflows/.github/workflows/pr-report.yml@v4.0.0
    permissions:
      pull-requests: write
```

The comment looks like this:

| Target | Result | Builder | Build step |
|---|---|---|---|
| `cpu` | success | alert-server | 13 s |
| `cuda` | success | alert-server | 14 s |

A run on a branch without an open pull request gets no comment. A pull request from a fork gets no comment, because its token is read-only. A failed comment does not turn the run red.

These conditions must stay true:

- The Tailscale trust credential has the subject `repo:*` and the custom claim `repository_owner_id` = `8375129`, and gives `tag:github-ci`. GitHub gives older repositories the subject `repo:ALERTua/<repository>:...` and newer ones the immutable subject `repo:ALERTua@8375129/<repository>@<id>:...`, so the credential matches the account number, not the account name.
- The tailnet policy gives `tag:github-ci` only `tcp:1234` on alert-server.
- buildkit on alert-server listens on `100.76.242.86:1234`.
