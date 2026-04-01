# Mealie Backup Retention Terraform Handoff

This document defines the Azure Blob lifecycle policy that should own retention for Mealie file backups.

## Goal

Keep retention out of the Kubernetes CronJob.

The Mealie file-backup job should only:

1. create a `tar.gz` archive from `/app-data`
2. upload it to Azure Blob Storage under `mealie/files/`

Azure Blob lifecycle management should delete old file backups by age.

## Target scope

Apply the lifecycle policy to:

- storage account: `sthomelabbackups`
- container: `mealie`
- prefix: `files/`

Equivalent full blob prefix:

- `https://sthomelabbackups.blob.core.windows.net/mealie/files/`

## Retention requirement

Delete Mealie file-backup blobs after **60 days**.

Rationale:

- the backup job runs weekly
- the prior in-job behavior kept `8` backups
- a 60-day age rule keeps roughly 8 weekly backups without requiring list-and-prune logic in Kubernetes

A nearby value in the **56-60 day** range is acceptable if Terraform or existing storage policy conventions prefer a different exact number, but the intended outcome is still "about eight weekly backups."

## Blob matching expectations

The rule should apply only to the Mealie file-backup path, not the CNPG database backup path.

Apply retention to:

- `files/`
- or a narrower prefix such as `files/mealie-app-data-backup-` if that better fits the Terraform implementation

Do **not** apply this rule to:

- `db/`
- unrelated containers or backup prefixes in the same storage account

## Operational contract

After this policy is in place:

- `homelab` should not reintroduce retention pruning in `apps/production/mealie/backup-script-configmap.yaml`
- the helper image should not implement delete-or-prune behavior for `mealie/files/`
- operators should treat Azure lifecycle management as the source of truth for file-backup retention

## Validation

Confirm the Terraform change does all of the following:

1. creates or updates a lifecycle rule on `sthomelabbackups`
2. scopes the rule to the `mealie/files/` backup path
3. deletes blobs after about 60 days
4. leaves `mealie/db/` retention untouched

## Related docs

- `apps/production/mealie/RUNBOOK.md`
- `apps/production/mealie/BACKUP_IMAGE_HANDOFF.md`
- `openspec/specs/mealie-backup-image-contract/spec.md`
