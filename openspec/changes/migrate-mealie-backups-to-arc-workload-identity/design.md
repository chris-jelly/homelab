## Context

The homelab now has an Arc-managed OIDC issuer and a documented workload identity contract, and the Azure-side Mealie identity resources already exist in `~/git/jellylabs-dsc/infrastructure/azure/homelab/`. The `feat/deploy-mealie` branch already contains the first Mealie app, CNPG, and backup manifests, but those manifests still use secret-based Azure Blob access for both database and file backups.

This is the first real workload rollout on top of the Arc foundation, so it needs to prove both halves of the design: database backups should use the CloudNativePG Azure identity path, and file backups should use the same federated identity instead of a storage connection string. The implementation base now exists, so the change can focus on converting those manifests instead of designing Mealie from scratch.

## Goals / Non-Goals

**Goals:**
- Bind Mealie backup consumers to the existing Mealie Azure workload identity resources while matching the actual Kubernetes service account subjects used at runtime.
- Remove the need for Mealie-specific Azure Blob credential secrets in Kubernetes and Azure Key Vault.
- Standardize Mealie database and file backups on the `db/` and `files/` prefixes in the `mealie` blob container.
- Update the Mealie rollout already present on `feat/deploy-mealie` rather than inventing a second manifest shape.
- Document validation and rollback steps for the first real workload identity consumer in the homelab.

**Non-Goals:**
- Reworking the Azure Arc platform setup or the shared `azure-creds` bootstrap secret for External Secrets.
- Migrating unrelated workloads such as Actual Budget, Linkding, or Audiobookshelf in the same change.
- Redesigning Mealie application deployment details that are unrelated to backup identity wiring.
- Changing the Azure-side identity shape that already exists in `~/git/jellylabs-dsc` unless rollout uncovers a concrete mismatch.

## Decisions

### Build on `feat/deploy-mealie` as the rollout base
The workload identity migration will apply to the existing Mealie manifests on `feat/deploy-mealie`, especially the backup resources in `apps/production/mealie/` and `databases/data/mealie/`.

Rationale:
- The Mealie deployment and backup structure already exist on that branch.
- Updating the existing branch avoids designing two competing Mealie rollout shapes.
- The branch already includes a backlog item for workload identity wiring, which this change now makes concrete.

Alternatives considered:
- Recreate the Mealie manifests independently on `main`: unnecessary duplication and higher merge risk.

### Use an explicit service account for the file-backup CronJob and the CNPG-generated service account for database backups
The `/app/data` backup CronJob will run as an explicit `ServiceAccount/mealie-backup` in namespace `mealie`. The CNPG cluster will continue to run with the operator-generated service account subject `system:serviceaccount:mealie:mealie-db-production-cnpg-v1`, and Azure will trust that second subject through an additional federated credential on the same managed identity.

Rationale:
- Runtime inspection showed that CNPG did not create or use `ServiceAccount/mealie-backup`; it generated and used `mealie-db-production-cnpg-v1`.
- The file-backup CronJob still benefits from a stable explicit service account name that matches the original Azure handoff contract.
- Adding a second federated credential is lower risk than trying to force CNPG onto an unsupported or uncertain custom service account path.

Alternatives considered:
- Force CNPG onto `ServiceAccount/mealie-backup`: appealing in theory, but inconsistent with observed CNPG runtime behavior in this cluster.
- Separate managed identities for CNPG and file backups: tighter separation, but unnecessary complexity for the first consumer rollout.

### Require webhook mutation through the standard pod label
Every Mealie backup workload that should receive Azure workload identity mutation will set `azure.workload.identity/use: "true"` on the pod template.

Rationale:
- This matches the Arc workload identity contract already documented in the bootstrap runbook.
- It makes the mutation path explicit instead of relying on cluster-wide assumptions.

Alternatives considered:
- Manual token projection without the pod label: possible, but diverges from the documented Arc path and adds per-workload complexity.

### Use secretless Azure Blob authentication for both backup paths
Mealie backup consumers will authenticate to Azure Blob through workload identity instead of connection strings, account keys, or SAS tokens stored in Kubernetes secrets.

Rationale:
- The Azure-side design already exists to avoid static backup credentials.
- This validates the main operational benefit of the Arc rollout.
- It prevents creating a Mealie-specific exception that would weaken the new pattern.

Alternatives considered:
- Keep a temporary connection-string secret for faster rollout: easier short term, but undermines the purpose of the new identity path.

### Use `azcopy` with workload identity for the file-backup job
The `/app/data` backup CronJob will use `ServiceAccount/mealie-backup`, and the backup client will switch from Azure CLI connection-string commands to `azcopy` with workload identity support.

Rationale:
- The user wants one federated Azure auth mechanism for all Mealie backup consumers.
- This proves the whole Mealie backup path can be secretless, not just the database half.
- It removes the need for `backup-externalsecret.yaml` and `mealie-backup-secrets` on the app side.
- AzCopy added workload identity support in the 10.25 release line, which makes it a better fit than forcing `az storage blob` commands to keep working without a connection string.

Alternatives considered:
- Keep the file-backup CronJob on a connection string while only migrating CNPG: simpler short term, but leaves the first consumer rollout only half converted.
- Keep Azure CLI as the Blob client and force it through a federated login flow: possible, but heavier than needed for a simple archive upload and retention job.

### Prefer the latest CNPG Azure identity path for database backups
If Mealie uses CloudNativePG, the backup configuration should target the latest supported CNPG and Barman plugin path and use `azureCredentials.useDefaultAzureCredentials: true`.

Rationale:
- The Azure-side handoff explicitly points to this path.
- It keeps CNPG on its native workload identity path while Azure trusts the actual CNPG service account subject.

Alternatives considered:
- Legacy CNPG secret-based Azure configuration: compatible with older examples, but inconsistent with the new Mealie backup contract.

### Standardize final Blob destinations on `mealie/db` and `mealie/files`
The Mealie CNPG backup path will target `https://sthomelabbackups.blob.core.windows.net/mealie/db/`, and the file-backup job will target `https://sthomelabbackups.blob.core.windows.net/mealie/files/`.

Rationale:
- These paths match the Azure-side handoff and Terraform outputs.
- They replace the older branch-local backup targets such as `cnpg-backups/mealie` with the workload-owned container model.
- They keep database and file artifacts separate without creating separate containers.

Alternatives considered:
- Preserve the existing branch path layout under `cnpg-backups/mealie`: workable, but inconsistent with the new one-container-per-workload Azure design.

## Risks / Trade-offs

- [Mealie manifests may not exist yet in the repo] -> Confirm the target paths early and pause if the workload deployment itself is still out of scope.
- [CNPG and file-backup consumers use different service account subjects] -> Keep one managed identity, but document the two federated subjects explicitly and validate both paths separately.
- [The first workload identity consumer may expose webhook or Azure client behavior that docs did not cover] -> Add validation steps for service account annotation, pod mutation, token claims, and Blob access before declaring rollout complete.
- [Rollback may need to preserve backups while removing identity wiring] -> Roll back the consuming workloads first, then remove service account annotations or backup config changes without deleting Azure storage data.

## Migration Plan

1. Move the implementation context onto `feat/deploy-mealie` or merge that branch into the working branch before editing Mealie manifests.
2. Add `ServiceAccount/mealie-backup` in namespace `mealie` with the Azure client ID annotation from the Terraform handoff.
3. Update the file-backup CronJob pod template to use `serviceAccountName: mealie-backup` and label `azure.workload.identity/use: "true"`.
4. Keep the CNPG cluster on the operator-generated service account subject and add an Azure federated credential for `system:serviceaccount:mealie:mealie-db-production-cnpg-v1`.
5. Replace the CNPG secret-based Azure backup config with the workload identity path and point it at `https://sthomelabbackups.blob.core.windows.net/mealie/db/`.
6. Replace the file-backup CronJob connection-string flow with `azcopy` using workload identity auth and point it at `https://sthomelabbackups.blob.core.windows.net/mealie/files/`.
7. Remove no-longer-needed Mealie backup credential secrets or ExternalSecrets.
8. Validate the CronJob and CNPG paths separately by checking their effective service account subjects and Blob authentication behavior.

Rollback strategy:
- Revert the Mealie service account annotation, pod labels, and backup configuration first.
- Restore the previous secret-based backup configuration only if one existed before this change.
- Leave the Azure-side container and identity in place unless there is a strong reason to remove them.

## Open Questions

- None at this time. The remaining work is implementation and validation of the corrected two-subject Azure federation model.
