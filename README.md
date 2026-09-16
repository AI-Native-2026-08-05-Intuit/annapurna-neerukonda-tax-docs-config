# taxdocs-config

GitOps config for taxdocs-api. Argo CD (v2.11.7) reconciles `overlays/*` onto k3d / the destination cluster.

- `base/` — W5 D3 workloads (Namespace, Deployment, Service, ConfigMap, ServiceMonitor)
- `overlays/{dev,staging,prod}` — namespace, replica, image tag, Spring profile
- `argocd/` — AppProject, the `taxdocs-api-dev` Application anchor, ApplicationSet
- `argocd-system/notifications-cm.yaml` — Slack on sync-failed and health-degraded only

The Slack token lives in `argocd-notifications-secret` (created out of band). Do not commit it.
