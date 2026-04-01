## 1. Define the backup handoff

- [x] 1.1 Add a Mealie backup handoff doc in `homelab` that describes the simplified backup script behavior, env contract, mounts, and workload identity expectations.
- [x] 1.2 Update the Mealie runbook to reference the new handoff docs and clarify that Azure lifecycle policy now owns file-backup retention.
- [x] 1.3 Add a Terraform handoff doc that requests Azure Blob lifecycle management for the `mealie/files/` prefix and documents the target retention window.

## 2. Simplify the Kubernetes backup logic

- [x] 2.1 Simplify `backup-script-configmap.yaml` so it only creates the archive and uploads it with AzCopy.
- [x] 2.2 Remove prune-related logic and runtime dependencies that exist only to support in-job retention.
- [x] 2.3 Simplify the Mealie backup CronJob/config manifests so they match the reduced script contract.

## 3. Validate the refactor

- [x] 3.1 Reconcile the Mealie app manifests and manually trigger the backup CronJob with the simplified script contract.
- [x] 3.2 Verify that the refactored CronJob still uploads to `mealie/files` with workload identity and no longer performs in-job retention pruning.
- [x] 3.3 Confirm the runbook and handoff docs match the final script contract and the Azure lifecycle retention model.
