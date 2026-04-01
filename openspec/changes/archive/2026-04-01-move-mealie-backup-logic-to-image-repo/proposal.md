## Why

The current Mealie file-backup CronJob proves that workload identity authentication and AzCopy upload work, but too much backup behavior still lives in Kubernetes ConfigMaps. The CronJob mounts a shell script from `backup-script-configmap.yaml`, the prune logic embeds inline Python, and runtime dependencies such as `python3` are now implicit contracts between `homelab`, `homelab-images`, and Azure storage.

That shape is awkward for this repo. `homelab` should declare workloads, schedules, identities, and destinations. A small inline shell wrapper is acceptable here, but complex backup behavior is not. Retention is a better fit for Azure storage lifecycle policy managed from Terraform, because the Mealie file-backup requirement is really age-based cleanup rather than complex application logic.

## What Changes

- Simplify the Mealie file-backup script so it handles only archive creation and AzCopy upload.
- Simplify the Mealie CronJob so it passes environment variables and mounts only the source PVC, script ConfigMap, and temporary workspace.
- Remove backup-specific ConfigMaps that exist only to support prune logic or indirection that is no longer needed.
- Add a handoff document in `homelab` that tells the image repo and operators what assumptions remain and what retention behavior has moved out to Terraform.
- Add a Terraform handoff for Azure Blob lifecycle management so retention for `mealie/files/` is enforced by storage policy instead of the backup job.
- Update the Mealie runbook to describe the new repo boundary: `homelab-images` owns backup logic, `homelab` owns deployment wiring.

## Capabilities

### New Capabilities
- `mealie-backup-image-contract`: Define the contract between the Mealie backup CronJob in `homelab` and the helper image published from `homelab-images`.

### Modified Capabilities
- `mealie-azure-backups`: Simplify the Kubernetes-side Mealie file-backup deployment so the local script handles only archive creation and upload while Azure storage lifecycle policy owns retention.

## Impact

- Affected code: `apps/production/mealie/`, Mealie runbook docs, and a new handoff doc for the image repo.
- Affected systems: the Mealie file-backup CronJob, Azure Blob lifecycle management for `sthomelabbackups`, and optionally the `ghcr.io/chris-jelly/mealie-backup-azcopy` image if image defaults need cleanup.
- Dependencies: the Mealie workload identity wiring stays in place; this change mainly simplifies backup-script ownership and moves retention ownership.
