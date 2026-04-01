# Mealie Backup Image Handoff

This document defines what the published `mealie-backup-azcopy` image must continue to provide for the Mealie file-backup CronJob.

## Goal

Keep the Mealie file-backup implementation small in `homelab` while relying on the published helper image for the runtime tools required to archive data and upload it with AzCopy.

## Expected image behavior

The published `mealie-backup-azcopy` image should:

1. Provide `bash`, `tar`, `gzip`, and `azcopy` in the image.
2. Support AzCopy workload identity login.
3. Work with the current non-root security context and writable `/tmp` mount.
4. Exit non-zero on upload failure.
5. Avoid implementing retention pruning in the container. Azure storage lifecycle policy will own retention for `mealie/files/`.

## Required runtime contract

The Mealie backup job uses these environment variables:

- `BACKUP_DESTINATION_URL`: Blob destination prefix, for example `https://sthomelabbackups.blob.core.windows.net/mealie/files`
- `BACKUP_PREFIX`: archive filename prefix, for example `mealie-app-data-backup`
- `AZCOPY_AUTO_LOGIN_TYPE`: should remain `WORKLOAD`
- `AZCOPY_JOB_PLAN_LOCATION`: writable path such as `/tmp/azcopy-plans`
- `AZCOPY_LOG_LOCATION`: writable path such as `/tmp/azcopy-logs`

The image may support additional optional knobs, but these are the required inputs expected by `homelab`.

## Kubernetes assumptions

`homelab` will provide:

- `ServiceAccount/mealie-backup`
- Pod label `azure.workload.identity/use: "true"`
- PVC mount at `/app-data`
- Writable temporary directory mounted at `/tmp`
- a small ConfigMap-hosted shell script that creates the archive and calls `azcopy copy`

The image should not assume root access. It must work with the current CronJob security context.

## Output expectations

On success, logs should clearly show:

- backup start
- archive path and size
- AzCopy workload identity login success or upload success
- uploaded blob name

On failure, logs should clearly show:

- which step failed
- AzCopy or filesystem error details

## Image contract notes

- The image should support archive creation and upload performed by the small shell wrapper in `homelab`.
- The image should not list or delete older blobs.
- The image should not require prune-specific dependencies such as `python3` for the Mealie backup job.
- `homelab` should be able to switch to a new immutable image tag without changing the runtime contract.
