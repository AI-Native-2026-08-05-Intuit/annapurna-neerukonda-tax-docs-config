# taxdocs-config

GitOps config for taxdocs-api. Argo CD (v2.11.7) reconciles `overlays/*` onto k3d / the destination cluster.

- `base/` — W5 D3 workloads (Namespace, Deployment, Service, ConfigMap, ServiceMonitor)
- `overlays/{dev,staging,prod}` — namespace, replica, image tag, Spring profile
- `argocd/` — AppProject, the `taxdocs-api-dev` Application anchor, ApplicationSet
- `cfn/` — CloudFormation (bootstrap IAM + artefact bucket first)
- `taxdocs-api/INFRA.md` — how to deploy and why Retain is on both policies
- `argocd-system/notifications-cm.yaml` — Slack on sync-failed and health-degraded only (no on-sync-succeeded)

The Slack token lives in `argocd-notifications-secret` (created out of band). Do not commit it.

Every Application uses `spec.project: taxdocs` and `resources-finalizer.argocd.argoproj.io`. The ApplicationSet matrix (`list` × `clusters` labelled `uptimecrew.example.internal/tier=workload`) owns `taxdocs-api-dev/staging/prod` with `preserveResourcesOnDeletion: true`.
