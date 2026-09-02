# K8s Actions

Reusable GitHub Actions workflows for deploying to Kubernetes clusters and a composite action for packaging Helm charts.

## Workflows

| Workflow | Method | Description |
|----------|--------|-------------|
| `k8s-deploy-manifest.yml` | kubectl apply | Deploy raw K8s manifest files or directories |
| `k8s-deploy-helm.yml` | helm upgrade --install | Deploy Helm charts with values and overrides |
| `actions/helm-package-push` | helm package + helm push | Composite action to package Helm charts and push to OCI registries |

## Prerequisites

- A Kubernetes cluster accessible from GitHub Actions runners (or via WireGuard VPN)
- A base64-encoded kubeconfig stored as a repo secret
- Manifest files or a Helm chart in your repo

### Encoding kubeconfig

```bash
cat ~/.kube/config | base64 | pbcopy  # macOS
cat ~/.kube/config | base64 -w 0      # Linux
```

Store the output as a repo secret named `KUBECONFIG`.

## Usage

### Manifest Deploy

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-manifest.yml@main
    with:
      manifest-path: "k8s/"
      namespace: production
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

### Manifest Deploy with VPN

```yaml
jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-manifest.yml@main
    with:
      manifest-path: "k8s/deployment.yaml"
      namespace: staging
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

### Helm Deploy

```yaml
jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
      namespace: production
      values-file: "charts/myapp/values-prod.yaml"
      set-values: |
        image.tag=${{ github.sha }}
        replicas=3
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

### Helm Deploy — Minimal

```yaml
jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

### Helm Package & Push to OCI

`helm-package-push` is a composite action, so registry authentication can be done in a previous step in the same job. The action only runs dependency update, lint, package, and push.

```yaml
name: Package Helm Chart

on:
  push:
    tags: ["v*"]

permissions:
  contents: read
  id-token: write
  packages: write

jobs:
  package:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-helm@v4
        with:
          version: v3.16.0

      # Do vendor-specific OIDC login here, for example:
      # - aws-actions/configure-aws-credentials@v4 + helm registry login to ECR
      # - azure/login@v2 + az acr login
      # - google-github-actions/auth@v2 + gcloud auth configure-docker

      - uses: NFUChen/cloud-actions/.github/actions/helm-package-push@main
        with:
          chart-path: "charts/myapp"
          registry: ghcr.io
          repository: ${{ github.repository_owner }}/charts
          chart-version: ${{ github.ref_name }}
          app-version: ${{ github.sha }}
```

## Inputs

### Manifest Workflow

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `manifest-path` | string | yes | — | Path to manifest file or directory |
| `namespace` | string | no | `default` | Target K8s namespace |
| `kubectl-version` | string | no | `latest` | kubectl version to install |

### Helm Deploy Workflow

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `chart-path` | string | yes | — | Path to Helm chart directory |
| `release-name` | string | yes | — | Helm release name |
| `namespace` | string | no | `default` | Target K8s namespace |
| `values-file` | string | no | `""` | Path to values file |
| `values-files` | string | no | `""` | Comma-separated values file paths, applied in order |
| `set-values` | string | no | `""` | Newline-separated `key=value` pairs for `--set` |
| `helm-version` | string | no | `latest` | Helm version to install |
| `deploy-mode` | string | no | `release` | `release` or `template` |

### Helm Package & Push Action

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `chart-path` | yes | — | Path to Helm chart directory |
| `registry` | yes | — | OCI registry host |
| `repository` | yes | — | Repository path inside the registry, without chart name |
| `chart-version` | no | `""` | Chart version override passed to `helm package --version` |
| `app-version` | no | `""` | App version override passed to `helm package --app-version` |
| `dependency-update` | no | `true` | Run `helm dependency update` before packaging |
| `lint` | no | `true` | Run `helm lint` before packaging |
| `push` | no | `true` | Run `helm push`; set `false` for package-only validation |
| `destination` | no | `dist` | Output directory for the packaged `.tgz` |

## Secrets

The deploy workflows use these secrets. The Helm package action does not handle registry login; do vendor-specific OIDC auth in a previous step in the same job.

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Full kubeconfig file content (base64 encoded) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |

## How It Works

```
Push to main
  └─► k8s-deploy-manifest runs
        ├─ WireGuard VPN (if configured)
        ├─ kubectl cluster-info (connectivity check)
        ├─ kubectl apply -f <path> -n <namespace>
        └─ writes result to job summary

  └─► k8s-deploy-helm runs
        ├─ WireGuard VPN (if configured)
        ├─ kubectl cluster-info (connectivity check)
        ├─ helm upgrade --install <release> <chart> -n <namespace>
        └─ writes result to job summary

  └─► caller job authenticates to OCI registry
        └─► actions/helm-package-push runs
              ├─ helm dependency update <chart>
              ├─ helm lint <chart>
              ├─ helm package <chart>
              └─ helm push <package> oci://<registry>/<repository>
```

## Pinning Tool Versions

The deploy workflows accept a version input for reproducibility. For `actions/helm-package-push`, set up Helm before calling the composite action.

```yaml
with:
  kubectl-version: "v1.31.0"   # manifest workflow
  helm-version: "v3.16.0"      # helm workflow
```

Default is `latest` for both.

## Helm Values

### Via values file

```yaml
with:
  values-file: "charts/myapp/values-prod.yaml"
```

### Via --set overrides

One `key=value` pair per line:

```yaml
with:
  set-values: |
    image.tag=abc1234
    replicas=3
    ingress.enabled=true
```

### Both together

```yaml
with:
  values-file: "charts/myapp/values-prod.yaml"
  set-values: |
    image.tag=${{ github.sha }}
```

`--set` values override values file entries, matching standard Helm behavior.
