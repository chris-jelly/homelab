## Context

The production apps layer currently includes app-local Cloudflare tunnel manifests, with Linkding owning a dedicated `cloudflared` deployment and ingress mapping in `apps/production/linkding/cloudflare.yaml`. This pattern duplicates edge runtime concerns at the app level and makes new public hostname onboarding inconsistent. The cluster already uses Traefik for ingress routing, so a shared edge tunnel that forwards into Traefik aligns with existing routing primitives.

Constraints:
- Keep GitOps ownership in-repo with Flux-managed manifests.
- Keep secrets sourced via External Secrets and Azure Key Vault.
- Avoid service disruption for existing `linkding.jellylabs.xyz` during migration.

Stakeholders:
- Platform/infra owner (shared edge tunnel lifecycle and DNS contract)
- Application owners (per-app ingress rules and service readiness)

## Goals / Non-Goals

**Goals:**
- Centralize Cloudflare tunnel runtime into one shared edge workload.
- Standardize app exposure through per-app `Ingress` resources managed in each app directory.
- Preserve existing public hostname behavior while enabling additional hosts such as Streamlit.
- Define clear ownership boundaries for DNS, shared tunnel, and app routing.

**Non-Goals:**
- Replacing Traefik with a different ingress controller.
- Introducing path-based multi-app routing policy as the default pattern.
- Reworking internal-only service discovery or non-public app networking.

## Decisions

### Decision: Place shared tunnel manifests under infrastructure configs
- Rationale: `apps/` entries represent individual applications, while the shared tunnel is platform edge infrastructure.
- Implementation direction: manage shared tunnel manifests under `infrastructure/configs/base/` and include them through the infrastructure kustomization tree.
- Alternatives considered:
  - `apps/production/edge/`: rejected to avoid mixing shared platform edge resources into app ownership boundaries.

### Decision: Use one shared tunnel deployment that forwards to Traefik
- Rationale: Reduces duplicated tunnel deployments and credentials while leveraging existing ingress routing controls.
- Alternatives considered:
  - Per-app tunnel deployments: rejected due to operational duplication and fragmented ownership.
  - Shared tunnel with direct service host mappings: rejected as default because central config churn grows with each app and weakens app-level ownership.

### Decision: Keep hostname routing in app-owned Ingress manifests
- Rationale: Each app controls its own host/path/TLS intent in its own directory, reducing platform bottlenecks.
- Alternatives considered:
  - Centralized cloudflared host mapping only: rejected because every hostname change requires shared edge config updates.

### Decision: Adopt explicit hostname DNS records pointing to one tunnel target
- Rationale: Explicit records provide safer governance and clearer lifecycle than wildcard-first DNS.
- Operational model: DNS records remain an operational runbook step for now rather than repo-managed manifests.
- Alternatives considered:
  - Wildcard DNS (`*.jellylabs.xyz`): deferred due to risk of accidental exposure and weaker review clarity.
  - Repo-managed DNS records now: deferred to a later iteration once DNS management patterns are standardized.

### Decision: Migrate Linkding from app-local cloudflared to ingress-based exposure
- Rationale: Validates migration path on an existing public app before scaling pattern to additional apps.
- Alternatives considered:
  - Keep legacy Linkding tunnel path indefinitely: rejected because it preserves two competing exposure models.

## Risks / Trade-offs

- [Shared edge tunnel outage affects multiple apps] -> Run multiple replicas, keep health probes, and document rollback to previous app-local tunnel manifests until migration is complete.
- [Ownership confusion between infra and app teams] -> Define and document a route ownership contract in the change artifacts and implementation tasks.
- [Ingress misconfiguration could break host routing] -> Require per-app ingress scenarios and post-deploy validation per hostname.
- [DNS drift outside GitOps workflows] -> Maintain explicit DNS inventory and checklist for Cloudflare record creation/update during rollout.

## Migration Plan

1. Create shared edge infrastructure manifests for cloudflared deployment, shared config, and tunnel credentials.
2. Add/verify Traefik-facing ingress definitions for Linkding and `sales-pipeline-pulse` (streamlit) apps.
3. Create or update Cloudflare DNS host records to point both hostnames to the shared tunnel target.
4. Validate hostname routing end-to-end after direct cutover (no phased canary required for this migration).
5. Remove Linkding app-local `cloudflare.yaml` and tunnel credential key usage after successful cutover.

Rollback:
- Re-enable Linkding app-local cloudflared deployment and DNS mapping if shared edge routing fails.
- Keep migration steps reversible until all hostname checks pass.

## Open Questions

- None at this time. Location, DNS ownership model, and migration strategy are now defined for implementation.
