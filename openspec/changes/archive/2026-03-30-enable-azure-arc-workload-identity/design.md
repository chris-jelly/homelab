## Context

The cluster already exposes Kubernetes service account OIDC discovery endpoints, but the issuer and JWKS URLs are internal-only and do not satisfy Microsoft Entra workload identity federation. The earlier design direction was to self-host a public OIDC issuer behind Cloudflare Tunnel, but current Azure Arc documentation offers a more direct path for self-managed k3s clusters: connect the cluster to Azure Arc, enable Arc-managed OIDC issuer support, enable the workload identity feature, and point k3s at the Arc-provided issuer URL.

This changes the problem from “publish our own public issuer” to “adopt Azure Arc as the cluster workload identity control plane for Azure-facing workloads.” That is a broader platform integration, but it removes the need to design and operate a custom public issuer endpoint.

Microsoft's current documentation is slightly inconsistent about feature labeling. The conceptual page title still includes a preview marker, while the current quickstart and how-to pages describe the feature as part of the normal Azure Arc Kubernetes workflow for supported distributions, including k3s. This change will follow the documented Azure Arc flow without asserting a stronger support claim than the docs consistently provide.

## Goals / Non-Goals

**Goals:**
- Connect the homelab k3s cluster to Azure Arc.
- Enable Arc-managed OIDC issuer support and workload identity support.
- Reconfigure k3s to mint service account tokens with the Arc-provided issuer URL.
- Document a repeatable onboarding, validation, and cleanup workflow.
- Make the resulting issuer and webhook contract available to downstream workloads such as Mealie backups.

**Non-Goals:**
- Building or operating a self-hosted public OIDC issuer.
- Migrating all workloads to workload identity in the same change.
- Replacing non-Azure authentication patterns that are unrelated to workload identity.
- Introducing Azure Arc extensions beyond what workload identity and cluster connection require.

## Decisions

### Use Azure Arc as the cluster workload identity foundation
The cluster will use Azure Arc-enabled Kubernetes as the platform mechanism for public OIDC issuer hosting and Azure workload identity integration.

Rationale:
- Azure Arc provides a documented path for k3s clusters to expose a public issuer URL without custom issuer hosting.
- This removes the need for Cloudflare Tunnel, a custom proxy, or static JWKS hosting just to satisfy Entra federation.
- The resulting issuer URL is designed to integrate directly with Azure managed identities and federated credentials.

Alternatives considered:
- Self-host a public OIDC issuer behind Cloudflare Tunnel: workable, but adds bespoke public exposure and issuer maintenance.

### Use `rg-jellyhomelab` in `Canada Central` for the Arc-connected cluster resource
The Azure Arc-connected cluster resource and related workload identity resources will live in `rg-jellyhomelab` in `Canada Central`.

Rationale:
- This keeps Arc-managed cluster identity resources with the rest of the homelab Azure footprint.
- It avoids introducing a second resource group for a platform capability that serves the existing homelab cluster.
- It fixes the Azure placement choice up front so runbooks and automation can use one known region.

Alternatives considered:
- Create a dedicated Arc resource group: cleaner separation on paper, but extra overhead for a small homelab environment.

### Connect the cluster with both OIDC issuer and workload identity enabled
The cluster will be connected to Azure Arc with `--enable-oidc-issuer` and `--enable-workload-identity` so Arc both hosts the issuer and installs the workload identity mutation path.

Rationale:
- The issuer alone is not enough; pods also need the mutation webhook behavior and well-known token projection flow.
- The documented Arc workflow treats these flags as the normal cluster-level enablement path.

Alternatives considered:
- Enable only the Arc OIDC issuer and manage token projection manually: possible, but adds unnecessary per-workload wiring and diverges from the documented cluster feature path.

### Capture Arc onboarding in the cluster bootstrap runbook
Azure Arc onboarding and workload identity enablement will be documented in `specs/001-cluster-bootstrap.md` as part of the cluster bootstrap path.

Rationale:
- Arc onboarding and k3s issuer reconfiguration are bootstrap and control-plane tasks, not normal Flux-managed manifests.
- The bootstrap runbook already documents manual prerequisites and cluster bring-up steps, so it is the right home for Arc connection and issuer setup.
- This lets the change start replacing the current manual Azure credential bootstrap pattern now, not just defer it.

Alternatives considered:
- Keep Arc onboarding only in change-local notes: too easy to lose after implementation.
- Treat Arc onboarding as pure GitOps: not realistic for `connectedk8s` operations and control-plane reconfiguration.

### Split implementation into manual operator tasks and repo-managed tasks
This change will explicitly separate manual operator tasks from agent or repo-managed tasks.

Rationale:
- Azure Arc connection, Azure CLI steps, live validation, and k3s control-plane changes are operational tasks against running infrastructure.
- Runbook updates, change-artifact updates, and downstream manifest changes remain repo work that can be implemented and reviewed through the normal agent workflow.
- This separation makes the implementation plan safer and clearer.

Alternatives considered:
- Keep one mixed task list: simpler to read at first, but too easy to blur live-cluster operations with repo edits.

### Start replacing the manual Azure credentials bootstrap step now
This change will start replacing the current manual Azure credentials bootstrap step in `specs/001-cluster-bootstrap.md` with workload identity-based bootstrap guidance where Arc and workload identity make that possible.

Rationale:
- The current bootstrap runbook explicitly depends on a manually created `azure-creds` secret for External Secrets Operator.
- Arc-managed workload identity should improve the bootstrap story, and the change should begin that migration in the same implementation window.
- Updating the bootstrap documentation at the same time avoids landing Arc as an isolated platform feature that operators must mentally stitch into the old secret-based process.

Alternatives considered:
- Only describe a future migration path: safer on paper, but leaves the bootstrap docs half-updated and delays the main operational benefit.

### Reconfigure k3s to use the Arc-provided issuer URL
After Arc exposes the issuer URL, k3s will set `service-account-issuer` to that Arc URL and restart so newly minted projected service account tokens use the correct public issuer.

Rationale:
- Arc hosts the public issuer, but k3s still needs to mint tokens with that issuer value.
- This keeps Kubernetes token minting aligned with what Microsoft Entra will trust.

Alternatives considered:
- Leave k3s on the internal issuer and rely only on Arc connection state: insufficient, because token issuer claims must match the trusted issuer.

### Treat Azure Arc as a platform dependency with explicit resource and cleanup costs
The change will explicitly account for the Arc agent footprint, provider registration, Azure-side resources, and cleanup process.

Rationale:
- The Arc quickstart documents cluster agent installation, Azure resource provider registration, and cluster cleanup commands.
- This is a bigger footprint than a manifest-only workload change and should be documented as such.

Alternatives considered:
- Treat Arc enablement as a trivial prerequisite: underestimates the operational cost and makes rollback harder.

## Risks / Trade-offs

- [Azure Arc adds agent footprint to a small homelab cluster] -> Validate resource headroom before rollout and document the expected Arc agent cost from the Microsoft quickstart.
- [The cluster becomes more dependent on Azure control-plane services] -> Limit the scope of this change to workload identity use cases that already depend on Azure resources and document cleanup steps.
- [Changing the k3s service-account issuer affects newly minted projected tokens] -> Validate fresh token claims after rollout and roll dependent workloads after the issuer change.
- [Arc onboarding or workload identity setup fails midway] -> Keep a stepwise procedure with verification after cluster connect, issuer enablement, issuer retrieval, k3s reconfiguration, and federated credential creation.
- [Documentation around feature status remains inconsistent] -> Avoid overstating support level in repo docs and cite the exact Arc quickstart and workload identity docs used for implementation.

## Migration Plan

1. Register the Azure resource providers required for Azure Arc-enabled Kubernetes in `rg-jellyhomelab` and `Canada Central`.
2. Connect the cluster to Azure Arc with `az connectedk8s connect`.
3. Enable Arc OIDC issuer and workload identity support on the connected cluster if they were not enabled at connect time.
4. Retrieve the Arc-managed issuer URL from `az connectedk8s show`.
5. Update k3s configuration so `service-account-issuer` uses the Arc-provided issuer URL, then restart k3s.
6. Verify the cluster connection, Arc agents, issuer URL, and fresh service account token `iss` claim.
7. Create managed identities and federated credentials for downstream workloads, starting with Mealie backups.
8. Update workload manifests to use Arc workload identity labels, annotations, and service accounts.
9. Update `specs/001-cluster-bootstrap.md` so the bootstrap flow starts replacing the current manual Azure credentials bootstrap step with the Arc-backed workload identity path.

Rollback strategy:
- Remove or disable downstream workload identity consumers first if needed.
- Restore the previous k3s service-account issuer configuration and restart k3s.
- Disable or disconnect Azure Arc if the cluster should return to a fully non-Arc state.
- Use `az connectedk8s delete` for cleanup so Azure-side resources and in-cluster agents are removed consistently.

## Open Questions

- Which parts of the current `azure-creds` bootstrap flow can be replaced immediately in this change, and which still depend on non-workload-identity consumers that must migrate later?
