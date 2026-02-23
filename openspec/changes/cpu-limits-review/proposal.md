# Proposal: Review and Remove CPU Limits

## Problem

CPU limits in Kubernetes use Linux CFS (Completely Fair Scheduler) throttling, which imposes a hard ceiling on CPU usage per 100ms period — even when the node has idle CPU available. This causes artificial latency spikes, request timeouts, and cascading slowdowns. The Kubernetes community increasingly recommends using CPU **requests** (which set CFS weighting for fair sharing) without CPU **limits**.

However, this cluster runs on fixed hardware (Raspberry Pi control plane + laptop worker nodes, ~16GB RAM each) with no autoscaling, so the tradeoffs differ from cloud environments.

## Current State

### Workloads WITH CPU limits (13 containers, 8 files)

| Workload | File | CPU Request | CPU Limit | Ratio |
|---|---|---|---|---|
| airflow/gitSync | `apps/base/airflow/release.yaml` | 50m | 100m | 2:1 |
| airflow/scheduler | `apps/base/airflow/release.yaml` | 200m | 1000m | 5:1 |
| airflow/apiServer | `apps/base/airflow/release.yaml` | 200m | 1000m | 5:1 |
| airflow/workers | `apps/base/airflow/release.yaml` | 200m | 1000m | 5:1 |
| actualbudget | `apps/base/actualbudget/release.yaml` | 200m | 500m | 2.5:1 |
| actualbudget-backup | `apps/production/actualbudget/backup-cronjob.yaml` | 100m | 500m | 5:1 |
| homepage | `apps/base/homepage/deployment.yaml` | 100m | 500m | 5:1 |
| linkding | `apps/base/linkding/deployment.yaml` | 100m | 500m | 5:1 |
| cloudflared (audiobookshelf) | `apps/production/audiobookshelf/cloudflare.yaml` | 100m | 500m | 5:1 |
| cloudflared (linkding) | `apps/production/linkding/cloudflare.yaml` | 100m | 500m | 5:1 |
| source-controller | `clusters/production/flux-system/gotk-components.yaml` | 50m | 1000m | 20:1 |
| kustomize-controller | `clusters/production/flux-system/gotk-components.yaml` | 100m | 1000m | 10:1 |
| helm-controller | `clusters/production/flux-system/gotk-components.yaml` | 100m | 1000m | 10:1 |
| notification-controller | `clusters/production/flux-system/gotk-components.yaml` | 100m | 1000m | 10:1 |

### Workloads WITHOUT CPU limits

- `monitoring/*` — no resource limits of any kind
- `databases/*` — no resource limits of any kind
- `infrastructure/*` — no resource limits of any kind

### No LimitRange or CPU-based ResourceQuota objects exist.

## Key Insight: CPU vs Memory

CPU is a **compressible** resource — the kernel can time-share it fairly. Memory is **incompressible** — if a pod exceeds available memory, something gets OOM-killed. This means:

- **Memory limits**: Keep them. They prevent node instability.
- **CPU limits**: Usually unnecessary. CPU requests provide fair scheduling via CFS weighting without hard throttling.

## Considerations for This Cluster

### Why removing limits is generally good
- Eliminates artificial throttling when CPU is available
- Cloudflared tunnels won't get laggy during traffic spikes
- Flux controllers won't slow down during large reconciliations
- Airflow scheduler/apiServer won't hit latency spikes

### Why this cluster needs more thought than cloud environments
- Fixed hardware, no autoscaling — a runaway pod can't be drained to a new node
- Airflow workers execute arbitrary DAGs — a misbehaving DAG could saturate a node
- Raspberry Pi control plane is resource-constrained

## Approaches Under Consideration

1. **Blanket removal** — Remove all CPU limits, keep requests, rely on CFS weighting
2. **Selective removal** — Remove limits from trusted/predictable workloads (Flux, cloudflared, homepage, linkding, actualbudget), keep on risky ones (Airflow workers)
3. **Selective removal + add missing limits** — Also consider whether monitoring/databases should get resource specs

## Open Questions

- Should Airflow workers keep a CPU limit as a guardrail against runaway DAGs?
- Should the monitoring stack (Prometheus, Grafana) get resource requests/limits added?
- Should databases (CNPG) get resource specs?
- Are the current CPU *request* values well-calibrated, or do those need review too?
- Would a development standards update be needed to codify the policy going forward?

## Status

Exploration in progress. No decision made yet.
