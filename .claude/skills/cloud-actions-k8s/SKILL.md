---
name: cloud-actions-k8s
description: Generate GitHub Actions caller workflows for Kubernetes deployment via kubectl manifests or Helm charts, and Helm chart packaging/publishing to OCI registries. Use this skill whenever the user mentions K8s deployment, Kubernetes CI/CD, kubectl apply, Helm deploy, Helm upgrade, helm package, helm push, chart publishing, deploying to a cluster, or wants to set up automated Kubernetes or Helm chart pipelines in GitHub Actions. Also use when the user asks about cloud-actions K8s workflows, container orchestration CI/CD, or deploying apps to Kubernetes.
---

# K8s Deploy Workflows and Helm Package Action

Generate caller workflow YAML for the `NFUChen/cloud-actions` K8s reusable workflows and Helm package composite action.

## Available Workflows / Actions

| Type | File | Method |
|------|------|--------|
| Manifest workflow | `k8s-deploy-manifest.yml` | `kubectl apply` -- deploy raw YAML manifests |
| Helm deploy workflow | `k8s-deploy-helm.yml` | `helm upgrade --install` -- deploy Helm charts |
| Helm package action | `.github/actions/helm-package-push` | `helm dependency update`, `helm lint`, `helm package`, `helm push` to OCI |

## Choosing the Right Workflow

- **Manifest**: User has raw K8s YAML files (deployments, services, configmaps, etc.)
- **Helm**: User has a Helm chart directory with `Chart.yaml`, templates, and values files

If the user isn't sure, ask what's in their repo -- a `charts/` directory points to Helm, a `k8s/` or `manifests/` directory with plain YAML points to manifest.

## Base URL

```
NFUChen/cloud-actions/.github/workflows/<workflow-file>@main
```

## Shared Secrets (both workflows)

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Full kubeconfig file content (base64 encoded) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |

The caller must base64-encode their kubeconfig:
```bash
cat ~/.kube/config | base64 | pbcopy  # macOS
cat ~/.kube/config | base64 -w 0      # Linux
```

## Manifest Workflow Reference

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `manifest-path` | string | yes | -- | Path to manifest file or directory |
| `namespace` | string | no | `default` | Target K8s namespace |
| `kubectl-version` | string | no | `latest` | kubectl version to install |

### Example

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
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

## Helm Workflow Reference

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `chart-path` | string | yes | -- | Path to Helm chart directory |
| `release-name` | string | yes | -- | Helm release name |
| `namespace` | string | no | `default` | Target K8s namespace |
| `values-file` | string | no | `""` | Path to a single values file (deprecated -- prefer `values-files`) |
| `values-files` | string | no | `""` | Comma-separated list of values file paths; applied in order, later overrides earlier |
| `set-values` | string | no | `""` | Newline-separated `key=value` pairs for `--set` |
| `helm-version` | string | no | `latest` | Helm version to install |
| `deploy-mode` | string | no | `release` | `release` (`helm upgrade --install`) or `template` (`helm template \| kubectl apply`) |

### Example -- Full

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
      namespace: production
      values-files: "charts/myapp/values.yaml,charts/myapp/values-prod.yaml"
      set-values: |
        image.tag=${{ github.sha }}
        replicas=3
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

### Example -- Minimal

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

## Helm Package & Push Action Reference

Use `.github/actions/helm-package-push` when the user wants to package a Helm chart and push it to an OCI registry. This is a composite action, not a reusable workflow, so registry authentication must happen in an earlier step in the same job. Do not add username/password secrets unless the user explicitly asks; vendor registries commonly use OIDC in previous steps.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `chart-path` | yes | -- | Path to Helm chart directory |
| `registry` | yes | -- | OCI registry host |
| `repository` | yes | -- | Repository path inside registry, without chart name |
| `chart-version` | no | `""` | Override for `helm package --version` |
| `app-version` | no | `""` | Override for `helm package --app-version` |
| `dependency-update` | no | `true` | Run `helm dependency update` |
| `lint` | no | `true` | Run `helm lint` |
| `push` | no | `true` | Run `helm push`; set false for package-only validation |
| `destination` | no | `dist` | Output directory for `.tgz` |

### Example -- Package and push after caller-managed auth

```yaml
jobs:
  package:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with:
          version: v3.16.0
      # Caller does OCI auth here (AWS/Azure/GCP/GHCR/etc.)
      - uses: NFUChen/cloud-actions/.github/actions/helm-package-push@main
        with:
          chart-path: "charts/myapp"
          registry: ghcr.io
          repository: ${{ github.repository_owner }}/charts
          chart-version: ${{ github.ref_name }}
          app-version: ${{ github.sha }}
```

## Common Patterns

### Helm Values Override

Multiple values files + `--set` overrides (later files override earlier; `--set` takes highest precedence):

```yaml
with:
  values-files: "charts/myapp/values.yaml,charts/myapp/values-prod.yaml"
  set-values: |
    image.tag=${{ github.sha }}
```

### Deploy Mode: Release vs Template

- **`release`** (default): runs `helm upgrade --install`. Helm tracks the release in cluster state (release secret). Use for normal lifecycle management, rollbacks via `helm rollback`.
- **`template`**: runs `helm template | kubectl apply`. No Helm release is created; manifests are applied as if they were raw YAML. Use for GitOps-style flows where you don't want Helm state in the cluster, or when migrating off Helm.

```yaml
with:
  chart-path: "charts/myapp"
  release-name: myapp
  deploy-mode: template
```

### Pinning Tool Versions

```yaml
with:
  kubectl-version: "v1.31.0"   # manifest workflow
  helm-version: "v3.16.0"      # helm workflow
```

### Docker Build + K8s Deploy Pipeline

A common pattern: build the image first, then deploy with the new tag.

```yaml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-ghcr.yml@main
    with:
      image-name: ghcr.io/${{ github.repository }}
      tags: |
        ${{ github.sha }}
      push: true

  deploy:
    needs: build
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
      namespace: production
      set-values: |
        image.tag=${{ github.sha }}
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

## Generation Guidelines

When generating caller workflows:

- Ask whether the user has raw manifests or a Helm chart if not clear
- Always ask for the namespace -- don't assume `default`
- Include `wg-config-file` only if user mentions VPN or private cluster
- For Helm deploy, always ask for the release name -- it's required and must be meaningful
- For Helm package/push, generate a normal job with steps; do not use `workflow_call` because prior auth steps must share the same runner
- For Helm package/push, assume the caller handles OCI registry auth before the action unless they explicitly ask for auth steps
- If user mentions Docker build + deploy, generate the combined pipeline pattern with `needs: build`
- Remind user to base64-encode their kubeconfig and store it as `KUBECONFIG` secret only for deploy workflows
