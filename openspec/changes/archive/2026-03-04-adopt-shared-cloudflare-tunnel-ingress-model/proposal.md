## Why

Public Cloudflare tunnel exposure is currently app-local, which duplicates tunnel runtime configuration and makes onboarding new public apps inconsistent. We need a shared, predictable edge pattern now to expose Streamlit and future services without coupling each app to its own tunnel workload.

## What Changes

- Introduce a shared Cloudflare tunnel deployment and configuration that acts as the single edge entrypoint for selected public hostnames.
- Route traffic from Cloudflare tunnel to the in-cluster ingress controller (Traefik), shifting hostname routing ownership to per-app `Ingress` resources.
- Remove app-local tunnel ownership for Linkding and replace it with app-owned ingress definitions for public routing.
- Define operational ownership and lifecycle rules for DNS records, shared tunnel config, and app ingress changes.

## Capabilities

### New Capabilities
- `shared-cloudflare-edge-routing`: Manage a centralized Cloudflare tunnel that forwards to cluster ingress while applications own hostname routing through standard ingress manifests.

### Modified Capabilities
- None.

## Impact

- Affected code: `apps/production/linkding/`, `apps/production/sales-pipeline-pulse/`, and a new shared edge directory under `infrastructure/configs/base/`.
- Affected systems: Cloudflare Tunnel runtime, Cloudflare DNS records, Traefik ingress routing, External Secrets for tunnel credentials.
- Operational impact: central edge ownership with per-app ingress ownership; DNS changes remain operational runbook steps for now; migration path needed for existing Linkding tunnel routing.
