# Mealie Backup Workload Identity Handoff

This package captures the Azure-side contract that the homelab repo should consume for Mealie backups.

## Azure resources

- Resource group: `rg-jellyhomelab`
- Storage account: `sthomelabbackups`
- Backup container: `mealie`
- User-assigned managed identity: `id-mealie-backup`
- Federated identity credential: `fic-mealie-backup`

## Kubernetes binding

- Namespace: `mealie`
- ServiceAccount: `mealie-backup`
- Federated subject: `system:serviceaccount:mealie:mealie-backup`
- OIDC issuer source: set `kubernetes_oidc_issuer_url` in this stack to the cluster's issuer URL before apply

The homelab repo should annotate the `mealie-backup` service account with the UAMI client ID and ensure backup-consuming pods carry the required workload identity label so Azure federation environment variables are injected.

## Backup destinations

- Container URL: `https://sthomelabbackups.blob.core.windows.net/mealie`
- CNPG database prefix: `https://sthomelabbackups.blob.core.windows.net/mealie/db/`
- File-backup prefix: `https://sthomelabbackups.blob.core.windows.net/mealie/files/`

Use stable `db/` and `files/` prefixes so backup tooling can change later without changing the storage namespace.

## CNPG guidance

- Target the latest supported CloudNativePG and Barman plugin versions in the homelab repo.
- Configure Azure Blob authentication with `azureCredentials.useDefaultAzureCredentials: true`.
- Bind CNPG backup consumers to the `mealie-backup` service account rather than syncing storage credentials from Key Vault.
- Scope Azure access through the `Storage Blob Data Contributor` role assignment on the `mealie` container.

## File-backup guidance

- Reuse the same `mealie-backup` service account for the `/app/data` backup job.
- Use Azure-aware tooling that can authenticate through workload identity rather than a storage connection string.
- Do not create backup credential secrets in Azure Key Vault or Kubernetes for Mealie Blob access.

## Useful outputs

After `tofu apply`, use these outputs to wire the homelab repo:

- `mealie_backup_uami_client_id`
- `mealie_backup_uami_principal_id`
- `mealie_backup_container_name`
- `mealie_backup_db_prefix_url`
- `mealie_backup_files_prefix_url`
