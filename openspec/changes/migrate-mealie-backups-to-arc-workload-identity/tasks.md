## 1. Rebase the Mealie rollout context

- [x] 1.1 Move the implementation context onto `feat/deploy-mealie` or merge that branch into the working branch so the Mealie manifests are available locally.
- [x] 1.2 Confirm the Mealie backup resources on that branch are the migration targets: the CNPG backup config in `databases/data/mealie/` and the file-backup CronJob in `apps/production/mealie/`.

## 2. Wire Kubernetes workload identity

- [x] 2.1 Add an explicit `ServiceAccount/mealie-backup` in namespace `mealie` with the Azure client ID annotation from the Terraform handoff.
- [x] 2.2 Update the file-backup CronJob pod template to use `serviceAccountName: mealie-backup` and label `azure.workload.identity/use: "true"`.

## 3. Migrate backup consumers off Blob secrets

- [x] 3.1 Update the Mealie database backup configuration to use the workload identity Azure path and target `https://sthomelabbackups.blob.core.windows.net/mealie/db/`.
- [x] 3.2 Update the Mealie file-backup job to use `azcopy` with workload identity auth instead of `AZURE_STORAGE_CONNECTION_STRING` and target `https://sthomelabbackups.blob.core.windows.net/mealie/files/`.
- [x] 3.3 Remove any Mealie-specific Blob credential secret or ExternalSecret resources that become obsolete after the workload identity rollout.

## 4. Validate and document rollout

- [x] 4.1 Update the Mealie validation steps to distinguish the CronJob service account subject from the CNPG runtime service account subject and confirm both are trusted by Azure.
- [x] 4.2 Document the rollback path for the corrected two-subject workload identity model if Blob access fails.

## 5. Align Azure federation with runtime subjects

- [x] 5.1 Update the Azure/Terraform handoff so the Mealie managed identity trusts `system:serviceaccount:mealie:mealie-backup` for the file-backup CronJob.
- [x] 5.2 Add a second federated credential for `system:serviceaccount:mealie:mealie-db-production-cnpg-v1` so CNPG WAL archiving and backups can authenticate.
- [x] 5.3 Validate that the file-backup CronJob starts successfully and that CNPG `ContinuousArchiving` becomes healthy after the Azure change lands.
