# Homelab

GitOps configuration for my homelab Kubernetes cluster. [Argo CD](https://argo-cd.readthedocs.io/) watches this repo (`main` branch) and keeps the cluster in sync with it, using the app-of-apps pattern.

## What's deployed

| App | Chart / source | Version | Namespace | Purpose |
|-----|----------------|---------|-----------|---------|
| Argo CD | `argo-cd` from [argo-helm](https://argoproj.github.io/argo-helm) | 10.1.3 | `argocd` | GitOps controller; manages itself (Dex SSO disabled) |
| MetalLB | `metallb` from [metallb.github.io](https://metallb.github.io/metallb) | 0.15.2 | `metallb-system` | Bare-metal LoadBalancer implementation |
| MetalLB config | plain manifests in `config/metallb` | n/a | `metallb-system` | Address pool and L2 advertisement |

### MetalLB network

- Pool `homelab-pool`: `192.168.49.200` – `192.168.49.250`
- Advertised in Layer 2 mode via `homelab-l2`

## Repo layout

```
bootstrap/
  bootstrap.yaml        # Root Application ("argo-objects"), applied once by hand
manifests/argocd/       # Argo CD Application objects (picked up by the root app)
  argocd.yaml           # Argo CD managing itself
  metallb.yaml          # MetalLB Helm chart (sync-wave 0)
  metallb-config.yaml   # MetalLB pool/advertisement (sync-wave 1)
helm/                   # Helm values overrides, one folder per chart
  argocd/values.yaml
  metallb/values.yaml
config/                 # Raw Kubernetes manifests deployed by Applications
  metallb/ipaddresspool.yaml
```

## How it works

1. `bootstrap/bootstrap.yaml` defines the root `argo-objects` Application. It points at `manifests/argocd/` and has automated sync with prune and self-heal.
2. Every Application in `manifests/argocd/` is created from that folder.
3. Helm-based apps use two sources: the upstream chart, plus this repo as `$values` so overrides come from `helm/<app>/values.yaml`.
4. Sync waves order MetalLB: the chart (wave 0) goes first, then its config (wave 1), so the CRDs exist before the `IPAddressPool` is applied. The config app also retries with backoff.

## Bootstrapping a new cluster

With Argo CD installed in the `argocd` namespace:

```sh
kubectl apply -f bootstrap/bootstrap.yaml
```

Argo CD then deploys everything else from this repo.

## Adding a new app

1. Helm chart: add an Application in `manifests/argocd/` using the two-source pattern, and put overrides in `helm/<app>/values.yaml`.
2. Plain manifests: put them in `config/<app>/` and add an Application in `manifests/argocd/` that points at that path.
3. Commit and push to `main`. Argo CD syncs it automatically.
