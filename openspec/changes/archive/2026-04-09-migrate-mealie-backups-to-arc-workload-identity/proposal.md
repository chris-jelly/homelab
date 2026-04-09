## Why

The Azure Arc workload identity foundation is now live, and the existing `feat/deploy-mealie` branch already includes Mealie app, database, and backup manifests. That branch still uses secret-based Azure Blob access for both CNPG and `/app/data` backups, so this change is needed to convert that in-flight Mealie rollout to the Arc-backed service account, pod-label, and federated identity contract.

Runtime validation exposed one incorrect assumption in the first implementation pass: CloudNativePG did not run with `ServiceAccount/mealie-backup`. It generated and used `ServiceAccount/mealie-db-production-cnpg-v1`, so Azure now needs to trust two Kubernetes subjects instead of one.

## What Changes

- Update the Mealie manifests on top of `feat/deploy-mealie` to consume the Azure resources already defined in `~/git/jellylabs-dsc/infrastructure/azure/homelab/`.
- Introduce an explicit `mealie-backup` service account annotated for Azure workload identity for the file-backup CronJob.
- Update Mealie database backup configuration to authenticate to Azure Blob Storage with workload identity instead of synced storage credentials and to target `https://sthomelabbackups.blob.core.windows.net/mealie/db/`.
- Update the Mealie file-backup job so it uses `azcopy` with federated workload identity and writes to `https://sthomelabbackups.blob.core.windows.net/mealie/files/` without a connection-string secret.
- Align the Azure federated identity setup with the actual runtime subjects used by the Mealie file backup job and the CNPG cluster.
- Document the Mealie-specific validation and rollback steps for the first end-to-end workload identity consumer in the homelab.

## Capabilities

### New Capabilities
- `mealie-azure-backups`: Define how Mealie backup workloads authenticate to Azure Blob Storage through the Arc-managed workload identity contract.

### Modified Capabilities
- None.

## Impact

- Affected code: Mealie manifests already staged on `feat/deploy-mealie`, including `apps/production/mealie/`, `databases/data/mealie/`, related Kustomizations, and Mealie runbook or change-local docs.
- Affected systems: Mealie backup workloads, CloudNativePG backup configuration for Mealie if present, Azure Blob Storage container `mealie`, and the Arc workload identity webhook path.
- Dependencies: the Arc-backed issuer and Mealie Azure identity resources must already exist and remain aligned with both the explicit `mealie-backup` CronJob service account subject and the CNPG-generated `mealie-db-production-cnpg-v1` service account subject.
