# setup-riptides

GitHub Action to install the [Riptides](https://riptides.io) daemon and join a control plane from a GitHub Actions runner. Authentication uses GitHub Actions OIDC, no join tokens or long-lived credentials required.

## What is Riptides?

Riptides is a zero-trust networking layer that runs as a kernel module on your hosts. For CI pipelines it solves two problems:

**Secure secret injection** — instead of storing cloud credentials, API keys, or service tokens in GitHub secrets, Riptides gives the runner a verified SPIFFE workload identity and enforces your policy at the network layer. Your CI job calls AWS, S3, internal APIs, or any other service exactly as it would in production — credentials are injected transparently based on the runner's identity, without ever touching a secret.

**Connection visibility** — every outbound and inbound TCP connection made during a CI job is tracked with full workload identity context: which workflow, which repository, which actor made the call, and whether it was allowed or denied by policy. This gives you the same traffic observability and access control in CI that you have across the rest of your fleet.

## Prerequisites

1. A Riptides control plane with a `GitHubActionsVerifier` scoped to your organisation:

```yaml
apiVersion: auth.riptides.io/v1alpha1
kind: Verifier
metadata:
  name: github-actions
spec:
  GitHubActions:
    audience: riptides                          # must match the action's audience input
  requiredMetadata:
    - githubactions:repository:owner: your-org  # restricts to your org (add more groups for more orgs)
```

   If you don't have the verifier set up yet, follow the [Connect GitHub Actions Runners](https://docs.riptides.io/guides/connect-github-actions/) guide for the full walkthrough.

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
```

### Fetch a cloud resource

No AWS access keys in GitHub secrets. Riptides injects temporary credentials based on the runner's workload identity.

```yaml
      - name: Fetch config from S3
        run: aws s3 cp s3://my-bucket/config.json ./config.json
```

### Post a deployment result

Riptides injects the bearer token for outbound calls to services in your policy, no secrets stored in the workflow.

```yaml
      - name: Notify Sentry of deployment
        run: |
          sentry-cli releases new "${{ github.sha }}"
          sentry-cli releases deploys "${{ github.sha }}" new -e production
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `controlplane-url` | yes | | URL of your Riptides control plane |
| `audience` | no | `riptides` | OIDC token audience, must match `GitHubActionsVerifier` config |
| `version` | no | `latest` | Daemon version to install |
| `wait-for-ready` | no | `true` | Wait for the daemon to be fully ready before the step finishes |
| `ready-timeout` | no | `120` | Seconds to wait for readiness |

## How it works

The action calls the Riptides [install.sh](https://docs.riptides.io/install.sh) with `--github-actions`. The installer:

1. Installs the kernel driver and daemon package
2. Calls `riptides daemon auth --plugin GitHubActions`, fetches an OIDC token from the Actions token endpoint and exchanges it for a SPIFFE x509 identity certificate
3. Starts the daemon as a systemd service
4. Waits until the driver reports the daemon fully ready — connected, trust anchors loaded, workload identity issued (`--wait-ready`)

Step 4 matters because the daemon needs a moment after the service starts before traffic is actually intercepted. Without it, a step running immediately after this action can open connections that are missed. Set `wait-for-ready: false` to skip the wait.

The runner VM is ephemeral so no cleanup step is needed.
