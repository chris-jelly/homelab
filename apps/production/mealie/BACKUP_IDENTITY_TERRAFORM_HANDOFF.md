# Mealie Backup Identity Terraform Handoff

This document defines the Azure workload identity contract that `jellylabs-dsc` must keep in sync with the Mealie backup manifests in `homelab`.

## Goal

Let both Mealie backup paths authenticate to Azure Blob Storage without a Kubernetes secret:

- the file-backup CronJob in `apps/production/mealie/`
- the CNPG cluster in `databases/data/mealie/`

Both workloads use the same Azure user-assigned managed identity, but they do **not** use the same Kubernetes service account subject at runtime.

## Required Azure resources

Create or confirm these Azure resources in `rg-jellyhomelab`:

- user-assigned managed identity: `id-mealie-backup`
- Blob container: `mealie` in storage account `sthomelabbackups`
- role assignment: `Storage Blob Data Contributor` scoped to the `mealie` container

## Required federated credentials

Create or confirm **two** federated credentials on `id-mealie-backup`.

### 1. File-backup CronJob subject

This federated credential covers the explicit Kubernetes service account used by the Mealie file-backup CronJob.

- suggested name: `fic-mealie-backup`
- subject: `system:serviceaccount:mealie:mealie-backup`

### 2. CNPG runtime subject

This federated credential covers the service account that CloudNativePG uses at runtime for the Mealie cluster.

- suggested name: `fic-mealie-db-production-cnpg-v1`
- subject: `system:serviceaccount:mealie:mealie-db-production-cnpg-v1`

Do not assume CNPG runs as `mealie-backup`. Runtime validation showed that CNPG uses its own generated service account, and Azure must trust that second subject for WAL archiving and base backups to work.

## OIDC issuer input

Set `kubernetes_oidc_issuer_url` in `~/git/jellylabs-dsc/infrastructure/azure/homelab/` to the Arc-managed issuer for this cluster before applying Terraform or OpenTofu.

Use the standard subject format:

```text
system:serviceaccount:<namespace>:<service-account>
```

## Outputs consumed by homelab

Keep these outputs available for the Kubernetes manifests and runbook:

- `mealie_backup_uami_client_id`
- `mealie_backup_uami_principal_id`
- `mealie_backup_container_name`
- `mealie_backup_db_prefix_url`
- `mealie_backup_files_prefix_url`

Expected destination prefixes:

- database backups: `https://sthomelabbackups.blob.core.windows.net/mealie/db/`
- file backups: `https://sthomelabbackups.blob.core.windows.net/mealie/files/`

## Validation

After the Azure change lands, confirm both workload subjects can authenticate:

1. Trigger a manual run of the `mealie-backup` CronJob and confirm AzCopy reports `Login with Workload Identity succeeded`.
2. Check `Cluster/mealie-db-production-cnpg-v1` and confirm the `ContinuousArchiving` condition becomes healthy.
3. If CNPG logs show `AADSTS700213`, re-check the federated credential subject for `mealie-db-production-cnpg-v1`.

## Related docs

- `apps/production/mealie/RUNBOOK.md`
- `apps/production/mealie/BACKUP_RETENTION_TERRAFORM_HANDOFF.md`
- `specs/001-cluster-bootstrap.md`
