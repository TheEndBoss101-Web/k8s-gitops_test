# k8s-gitops_test

ArgoCD GitOps repo for the homelab Kubernetes cluster — 3-node Talos Linux stack with MetalLB, Longhorn, Traefik, and a growing set of self-hosted services.

## Stack

| Layer | What | How |
|-------|------|-----|
| **Cluster** | Talos Linux | 1 control plane + 2 workers (10.0.10.x) |
| **GitOps** | ArgoCD | App of Things pattern — one chart to rule them all |
| **Ingress** | Traefik | L7 routing with multiple entrypoints |
| **Storage** | Longhorn | Distributed block storage, 2 replicas, default StorageClass |
| **LB** | MetalLB | L2 mode, IP pool `10.0.10.200-240` |
| **Repo** | GitHub | Bot pushes, human reviews, ArgoCD auto-syncs |

## Apps

| App | Type | Namespace | Notes |
|-----|------|-----------|-------|
| `traefik` | Helm chart | `traefik` | L7 ingress controller |
| `longhorn` | Helm chart | `longhorn-system` | Distributed block storage |
| `metallb` | Helm chart | `metallb-system` | LoadBalancer controller |
| `metallb-config` | Git (kustomize) | `metallb-system` | IP pool + L2 advertisement |
| `whoami` | Git (kustomize) | `default` | Test/debug pod |
| `gitlab` | Git (kustomize) | `gitlab` | Self-hosted GitLab EE |
| `postgresql` | Git (kustomize) | `database` | Postgres 16 for stateful apps |
| `vaultwarden` | Git (kustomize) | `vaultwarden` | Bitwarden-compatible password mgr |

## Bootstrap from bare Talos

Run these in order after `talosctl kubeconfig` is working.

### 1. ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to be ready, then grab the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

Wait for the `metallb-system` controller and speaker pods, then apply the IP pool and L2 advertisement from this repo:

```bash
kubectl apply -k ./metallb-config/
```

### 3. Longhorn

Requires the `longhorn-system` namespace to run privileged pods — Talos enforces PodSecurity baseline by default:

```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system pod-security.kubernetes.io/enforce=privileged
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/longhorn.yaml
```

> ⚠️ **Skip the label step and Longhorn will fail** with PodSecurity violations on Talos.

### 4. Bootstrap ArgoCD apps

Once all three are healthy, point ArgoCD at this repo:

```bash
kubectl apply -n argocd -k ./
```

Or click "NEW APP" in the ArgoCD UI and point it at the `apps/` Helm chart in this repo.

---

## Architecture

All apps are registered in [`apps/values.yaml`](./apps/values.yaml) as ArgoCD Application entries. The `apps/` Helm chart renders them into Application CRDs via a single template — add a line, ArgoCD picks it up.

```
k8s-gitops_test/
├── apps/                    # ArgoCD App of Things
│   ├── Chart.yaml
│   ├── values.yaml          # ← register apps here
│   └── templates/
│       └── applications.yaml
├── traefik/                 # Helm values for Traefik
├── longhorn/                # Helm values for Longhorn
├── metallb/                 # Helm values for MetalLB
├── metallb-config/          # MetalLB IP pool + L2 CRDs
├── whoami/                  # Simple debug app
├── gitlab/                  # GitLab manifests
├── postgresql/              # Postgres manifests
└── vaultwarden/             # Vaultwarden manifests
```

## Adding a new app

Two patterns — pick one:

### Git-based app (plain manifests)

1. Create a folder with a `kustomization.yaml` and your manifests
2. Add to `apps/values.yaml`:

```yaml
applications:
  - name: your-app
```

No path needed — defaults to the folder name at the repo root.

### External Helm chart

```yaml
applications:
  - name: your-chart
    namespace: argocd
    destination:
      namespace: your-ns
    repoURL: https://charts.example.com
    chart: your-chart
    targetRevision: 1.0.0
    helm:
      valueFiles:
        - https://raw.githubusercontent.com/TheEndBoss101-Web/k8s-gitops_test/main/your-chart/values.yaml
      values: |
        fullnameOverride: your-chart
```

## Convention notes

- **Secrets** use placeholders (`CHANGE_ME_*`) — generate real values before deploying
- **Cross-namespace DNS** follows `service.namespace.svc.cluster.local` pattern
- **IngressRoute** hostnames use `*.theendboss101.com` — point DNS to the MetalLB IP
- **PR workflow**: bot branch → push → PR → review → merge → ArgoCD auto-syncs