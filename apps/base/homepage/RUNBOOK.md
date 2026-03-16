# Homepage service catalog

This dashboard follows the `homepage-service-catalog` OpenSpec change.

## Current catalog inventory

| Workload | Representation | Notes |
|----------|----------------|-------|
| Homepage | Excluded workload | Homepage is the dashboard and does not render a self-tile. |
| Linkding | Ingress-backed application | Discovered from `apps/production/linkding/ingress.yaml`. |
| Sales Pipeline Pulse | Ingress-backed application | Discovered from `apps/production/sales-pipeline-pulse/ingress.yaml`. |
| Airflow | Ingress-backed application | Represented through the chart-managed API server ingress only. |
| Actual Budget | Ingress-backed application | Represented through the chart-managed ingress. |
| Audiobookshelf | Excluded workload | Production overlay is disabled; if reintroduced, prefer a local-only ingress. |
| Grafana | Curated observability entry | Keep manual entry to use internal service URLs and observability grouping. |
| Prometheus | Curated observability entry | Keep manual entry to use internal service URLs and observability grouping. |
| Flux | Curated platform service | Manual entry with a Flux-wide pod selector instead of the broken native widget. |
| CloudNative-PG | Curated platform service | Manual operator entry; avoid misleading app widget assumptions. |
| cert-manager | Curated platform service | Manual operator entry with Kubernetes app labels. |
| external-secrets | Curated platform service | Manual operator entry with Kubernetes app labels. |

## Local-only ingress convention

Internal web apps that should appear in Homepage but do not need public routing should use `<app>.local` as the ingress hostname and carry the same `gethomepage.dev/*` annotations as public ingresses.

## Review checklist

- Decide whether the workload belongs in Homepage.
- Use ingress annotations for Homepage-worthy ingress-backed apps.
- Add or update a curated Homepage entry when the app has no ingress.
- Remove manual entries when an app moves to ingress discovery.
- Intentionally exclude helper or backing workloads that are not operator-facing.
