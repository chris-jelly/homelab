## Why

The cluster needs a Kubernetes-to-Azure workload identity path for Mealie backups and likely future Azure-integrated workloads. A self-hosted public OIDC issuer is possible, but it adds custom public exposure, proxying, and issuer-hosting work that Azure Arc can replace with a documented Arc-managed issuer flow for k3s.

The current cluster already issues service account tokens, but its default issuer is internal-only and cannot be used directly with Microsoft Entra token federation. Azure Arc provides a managed way to connect the cluster, generate a public issuer URL, install the workload identity webhook, and standardize the downstream identity contract.

## What Changes

- Connect the production k3s cluster to Azure Arc.
- Enable Azure Arc OIDC issuer support and Azure Arc workload identity support for the cluster.
- Place the Arc-connected cluster resource in `rg-jellyhomelab` in `Canada Central`.
- Reconfigure k3s so new projected service account tokens use the Arc-provided issuer URL.
- Document the Azure Arc onboarding, validation, rollback, and cleanup workflow for this homelab in the cluster bootstrap runbook.
- Start replacing the current manual Azure credentials bootstrap step with workload identity-based bootstrap guidance.
- Update downstream workload-identity consumers, starting with Mealie backups, to consume the Arc-provided issuer contract instead of a self-hosted issuer design.

## Capabilities

### New Capabilities
- `azure-arc-workload-identity`: Provide an Arc-managed OIDC issuer and workload identity foundation that lets cluster workloads federate to Microsoft Entra without storage secrets or self-hosted issuer infrastructure.

### Modified Capabilities
- None.

## Impact

- Affected code: cluster bootstrap and operational docs, Azure onboarding instructions, k3s control-plane configuration guidance, and downstream workload identity manifests and runbooks.
- Affected systems: Azure Arc, Microsoft Entra workload identity federation, k3s service account token issuance, Azure CLI `connectedk8s` workflow, and the Arc agents installed in the cluster.
- Operational impact: this adds Azure Arc agents to the cluster, requires Azure resource/provider setup in `Canada Central`, changes the service-account issuer for newly minted tokens, updates the bootstrap path in `specs/001-cluster-bootstrap.md`, and starts replacing the current manual Azure credentials bootstrap step with workload identity-based setup.
