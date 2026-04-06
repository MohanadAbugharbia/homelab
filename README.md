# Homelab

My personal homelab running on Kubernetes (k3s), managed via GitOps with ArgoCD.

## Services

| Service | Namespace | Description |
|---|---|---|
| [Immich](https://immich.app) | `immich` | Self-hosted photo and video library |
| [Grafana](https://grafana.com) | `monitoring` | Dashboards and visualization |
| [Prometheus](https://prometheus.io) | `monitoring` | Metrics collection and alerting |
| [Longhorn](https://longhorn.io) | `longhorn-system` | Distributed block storage |
| [CNPG](https://cloudnative-pg.io) | `cnpg-system` | CloudNativePG PostgreSQL operator |
| [cert-manager](https://cert-manager.io) | `cert-manager` | Automated TLS certificate management |
| [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) | `sealed-secrets` | Encrypted secrets safe to store in git |
| [stakater-reloader](https://github.com/stakater/Reloader) | `stakater` | Automatic pod restarts on ConfigMap/Secret changes |

## Structure

```
argo-services/          # ArgoCD Application manifests + Helm values per service
├── cert-manager/
├── cnpg/
├── grafana/
├── immich/
├── longhorn/
├── prometheus/
├── sealed-secrets/
└── stakater-reloader/
```

Each service directory contains an ArgoCD `Application` manifest and the Helm values or raw Kubernetes manifests for that service. ArgoCD watches this repo and automatically syncs changes to the cluster.

## GitOps

All services are managed declaratively. ArgoCD is configured with `automated` sync and `selfHeal: true`, meaning:
- Pushing to `main` automatically deploys changes
- Manual changes to the cluster are reverted back to what's in git
