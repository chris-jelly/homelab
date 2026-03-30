## 1. Rebase the Mealie rollout context

- [x] 1.1 Move the implementation context onto `feat/deploy-mealie` or merge that branch into the working branch so the Mealie manifests are available locally.
- [x] 1.2 Confirm the Mealie backup resources on that branch are the migration targets: the CNPG backup config in `databases/data/mealie/` and the file-backup CronJob in `apps/production/mealie/`.

## 2. Wire Kubernetes workload identity

- [x] 2.1 Add `ServiceAccount/mealie-backup` in namespace `mealie` with the Azure client ID annotation from the Terraform handoff.
- [x] 2.2 Update each Mealie backup consumer pod template to use `serviceAccountName: mealie-backup` and label `azure.workload.identity/use: "true"`.

## 3. Migrate backup consumers off Blob secrets

- [x] 3.1 Update the Mealie database backup configuration to use the workload identity Azure path and target `https://sthomelabbackups.blob.core.windows.net/mealie/db/`.
- [x] 3.2 Update the Mealie file-backup job to use `azcopy` with workload identity auth instead of `AZURE_STORAGE_CONNECTION_STRING` and target `https://sthomelabbackups.blob.core.windows.net/mealie/files/`.
- [x] 3.3 Remove any Mealie-specific Blob credential secret or ExternalSecret resources that become obsolete after the workload identity rollout.

## 4. Validate and document rollout

- [x] 4.1 Document the Mealie-specific validation steps for service account annotation, pod mutation, token claims, and Blob access.
- [x] 4.2 Document the rollback path for the Mealie consumer rollout if workload identity wiring or Blob access fails.
