# ci-workflows

Shared GitHub Actions workflows for the repositories of ALERTua.

## pick-runner

`pick-runner.yml` chooses where a job runs. If the multirunner runner on alert-server is available, the job runs there. If not, the job runs on a GitHub-hosted runner.

The workflow joins the tailnet with the GitHub OIDC token through Tailscale workload identity federation. No repository needs a secret. Pull requests from forks get no OIDC token, so they always use a GitHub-hosted runner.

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

These conditions must stay true:

- The Tailscale trust credential trusts the subject `repo:ALERTua/*` and gives `tag:github-ci`.
- The tailnet policy gives `tag:github-ci` only `tcp:8152` on alert-server.
- multirunner on alert-server publishes `/metrics` on port 8152.
