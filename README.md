# setup-riptides

GitHub Action to install the [Riptides](https://riptides.io) daemon and join a control plane from a GitHub Actions runner. Authentication uses GitHub Actions OIDC — no join tokens or long-lived credentials required.

## Prerequisites

1. A Riptides control plane with a `GitHubActionsVerifier` configured for your repository owner:

```yaml
apiVersion: auth.riptides.io/v1alpha1
kind: Verifier
metadata:
  name: github-actions
spec:
  GitHubActions:
    repositoryOwner: your-org   # required — restricts to your org
    audience: riptides            # must match the action's audience input
```

2. The workflow must have `id-token: write` permission.

## Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required for OIDC token
      contents: read
    steps:
      - uses: riptideslabs/setup-riptides@v1
        with:
          controlplane-url: https://abc123.console.riptides.io

      # From here, outbound connections are transparently mTLS via Riptides
      - run: curl https://my-internal-service.example.com
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `controlplane-url` | yes | — | URL of your Riptides control plane |
| `audience` | no | `riptides` | OIDC token audience — must match `GitHubActionsVerifier` config |
| `version` | no | `latest` | Daemon version to install |

## How it works

The action calls the Riptides [install.sh](https://docs.riptides.io/install.sh) with `--github-actions`. The installer:

1. Installs the kernel driver and daemon package
2. Calls `riptides daemon auth --plugin GitHubActions` — fetches an OIDC token from the Actions token endpoint and exchanges it for a SPIFFE x509 identity certificate
3. Starts the daemon as a systemd service

The runner VM is ephemeral so no cleanup step is needed.
