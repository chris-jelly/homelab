# 002 - Data Resilience

A plan to make every stateful workload in the cluster recoverable from a full rebuild, using CloudNative-PG continuous backups for PostgreSQL-backed services and scheduled CronJob backups for file-based applications.

## Current State

| App | State Type | Storage | Backed Up? |
|-----|-----------|---------|------------|
| Audiobookshelf | Files (SQLite + media) | 4 PVCs (22 Gi) | No |
| Linkding | Files (SQLite) | 1 PVC (1 Gi) | No |
| Actual Budget | Files (SQLite) | 1 PVC (20 Gi, Helm) | Yes (Azure Blob, weekly) |
| Airflow | PostgreSQL (CNPG) | 2 Gi CNPG volume | No |
| Homepage | Stateless (ConfigMap) | None | N/A |

### Databases (CloudNative-PG)

| Cluster | Namespace | Storage | Backup? | Replicas |
|---------|-----------|---------|---------|----------|
| `airflow-db-production-cnpg-v1` | `airflow` | 2 Gi | No | 1 |
| `warehouse-db-production-cnpg-v1` | `warehouse` | 1 Gi | No | 1 |
| `warehouse-db-dev-cnpg-v1` | `warehouse-dev` | 1 Gi | No | 1 |

All three CNPG clusters run single-instance with no backup configuration, no WAL archiving, and no scheduled base backups. A node failure or cluster rebuild means total data loss.

## Goals

1. All CNPG databases continuously archive WAL to Azure Blob Storage and take scheduled base backups, enabling point-in-time recovery (PITR)
2. CNPG clusters can bootstrap from their Azure backup on a fresh cluster rebuild
3. File-based apps (Audiobookshelf, Linkding) have scheduled backups to Azure Blob Storage
4. Actual Budget's existing backup pattern is the template for other file-based backups
5. `warehouse-dev` is explicitly excluded — dev data doesn't need backup

## Strategy

### Tier 1: CloudNative-PG Continuous Backup (PostgreSQL services)

CNPG has native support for continuous backup via Barman to Azure Blob Storage. This gives us:

- **Continuous WAL archiving** — every transaction is shipped to object storage
- **Scheduled base backups** — full database snapshots on a schedule
- **Point-in-time recovery** — restore to any point between the oldest base backup and the latest WAL
- **Bootstrap from backup** — a new CNPG cluster can initialize from an existing backup, so a full cluster rebuild recovers automatically

This applies to:
- `airflow-db-production-cnpg-v1`
- `warehouse-db-production-cnpg-v1`

### Tier 2: CronJob Backup (file-based apps)

For apps that use embedded SQLite or file-based storage, follow the pattern already established by Actual Budget:

- A CronJob mounts the app's PVC read-only
- A bash script creates a timestamped `.tar.gz` archive
- The archive is uploaded to Azure Blob Storage via `az storage blob upload`
- Old backups are pruned to retain only the last N copies

This applies to:
- Audiobookshelf (config + metadata PVCs — media files are likely replaceable and very large, so only back up config/metadata)
- Linkding (data PVC containing the SQLite database)

Restore is manual: download the archive from Azure Blob, extract into the PVC.

## Implementation Plan

### Phase 1: CNPG Continuous Backups

#### 1a. Create Azure Blob Storage credentials secret

CNPG authenticates to Azure Blob Storage independently from External Secrets Operator. A secret with the storage account connection string or SAS token needs to exist in each database namespace.

Use an `ExternalSecret` to pull the Azure Storage credentials from Key Vault (the same pattern as Actual Budget's `backup-externalsecret.yaml`):

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cnpg-azure-backup-creds
  namespace: <database-namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: azure-kv-store
  target:
    name: cnpg-azure-backup-creds
  data:
    - secretKey: AZURE_STORAGE_ACCOUNT
      remoteRef:
        key: cnpg-backup-storage-account
    - secretKey: AZURE_STORAGE_KEY
      remoteRef:
        key: cnpg-backup-storage-key
    - secretKey: AZURE_STORAGE_SAS_TOKEN
      remoteRef:
        key: cnpg-backup-storage-sas-token
```

Decide between storage key and SAS token based on your Azure setup. SAS tokens are more scoped but expire.

#### 1b. Add backup configuration to CNPG Cluster resources

Add a `backup` stanza to each production CNPG Cluster definition in `databases/data/`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: airflow-db-production-cnpg-v1
  namespace: airflow
spec:
  # ... existing spec ...

  backup:
    barmanObjectStore:
      destinationPath: "https://<STORAGE_ACCOUNT>.blob.core.windows.net/cnpg-backups/airflow"
      azureCredentials:
        storageAccount:
          name: cnpg-azure-backup-creds
          key: AZURE_STORAGE_ACCOUNT
        storageKey:
          name: cnpg-azure-backup-creds
          key: AZURE_STORAGE_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
    retentionPolicy: "30d"
```

#### 1c. Create ScheduledBackup resources

Add a `ScheduledBackup` CR alongside each database definition:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: airflow-db-scheduled-backup
  namespace: airflow
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM UTC
  backupOwnerReference: self
  cluster:
    name: airflow-db-production-cnpg-v1
```

#### 1d. Enable bootstrap from backup

For disaster recovery, the Cluster definition should support bootstrapping from the Azure backup. This is activated by adding a `bootstrap` stanza. On a fresh cluster rebuild, CNPG will pull the latest base backup and replay WAL to recover:

```yaml
spec:
  bootstrap:
    recovery:
      source: airflow-db-backup

  externalClusters:
    - name: airflow-db-backup
      barmanObjectStore:
        destinationPath: "https://<STORAGE_ACCOUNT>.blob.core.windows.net/cnpg-backups/airflow"
        azureCredentials:
          storageAccount:
            name: cnpg-azure-backup-creds
            key: AZURE_STORAGE_ACCOUNT
          storageKey:
            name: cnpg-azure-backup-creds
            key: AZURE_STORAGE_KEY
        wal:
          compression: gzip
```

> **Important:** The `bootstrap.recovery` stanza should only be present when recovering. On normal operation, remove it (or use a Kustomize overlay) so CNPG initializes a fresh database on first deploy but recovers from backup on rebuilds. One approach is to keep the `externalClusters` reference always present and only toggle the `bootstrap` section when needed.

### Phase 2: File-Based App Backups

Follow the established Actual Budget pattern. Each app needs:

1. **ExternalSecret** — pulls Azure Storage connection string from Key Vault
2. **ConfigMap (config)** — container name, retention count, backup prefix
3. **ConfigMap (script)** — the backup bash script
4. **CronJob** — mounts the app PVC read-only, runs the script

#### 2a. Linkding

| Config | Value |
|--------|-------|
| PVC to back up | `linkding-data-pvc` |
| Mount path | `/data` |
| Azure container | `linkding-backups` |
| Backup prefix | `linkding-backup` |
| Schedule | `0 3 * * 0` (weekly, Sunday 3 AM) |
| Retention | 8 |

The backup script is identical to Actual Budget's — tar + gzip the data directory, upload to Azure Blob, prune old backups.

Files to create in `apps/production/linkding/`:
- `backup-externalsecret.yaml`
- `backup-config-configmap.yaml`
- `backup-script-configmap.yaml`
- `backup-cronjob.yaml`

Update `apps/production/linkding/kustomization.yaml` to include the new files.

#### 2b. Audiobookshelf

Only the config and metadata PVCs need backup. The audiobooks and podcasts PVCs contain media files that are large, likely replaceable (re-downloaded from source), and impractical to archive to blob storage regularly.

| Config | Value |
|--------|-------|
| PVCs to back up | `audiobookshelf-config-pvc`, `audiobookshelf-metadata-pvc` |
| Mount paths | `/config`, `/metadata` |
| Azure container | `audiobookshelf-backups` |
| Backup prefix | `audiobookshelf-backup` |
| Schedule | `0 3 * * 0` (weekly, Sunday 3 AM) |
| Retention | 8 |

The backup script needs a minor modification from the Actual Budget template to tar both directories into a single archive.

Files to create in `apps/production/audiobookshelf/`:
- `backup-externalsecret.yaml`
- `backup-config-configmap.yaml`
- `backup-script-configmap.yaml`
- `backup-cronjob.yaml`

Update `apps/production/audiobookshelf/kustomization.yaml` to include the new files.

### Phase 3: Azure Key Vault Prerequisites

Before any of the above works, the following secrets need to exist in Azure Key Vault (`kv-jellyhomelabprod`):

| Key Vault Secret Name | Used By | Value |
|------------------------|---------|-------|
| `cnpg-backup-storage-account` | CNPG backup ExternalSecrets | Azure Storage account name |
| `cnpg-backup-storage-key` | CNPG backup ExternalSecrets | Azure Storage account key |
| `cnpg-backup-storage-sas-token` | CNPG backup ExternalSecrets (alternative to key) | SAS token scoped to the backup container |
| `linkding-backup-connection-string` | Linkding backup ExternalSecret | Azure Storage connection string |
| `audiobookshelf-backup-connection-string` | Audiobookshelf backup ExternalSecret | Azure Storage connection string |

The Actual Budget backup already uses `actualbudget-backup-connection-string` from the same vault.

Also create the Azure Blob Storage containers:
- `cnpg-backups` (with virtual directories per database)
- `linkding-backups`
- `audiobookshelf-backups`
- `actualbudget-backups` (already exists)

## File Changes Summary

```
databases/data/
  airflow/
    database.yaml                    # Add backup stanza
    scheduled-backup.yaml            # New: ScheduledBackup CR
    backup-externalsecret.yaml       # New: Azure creds from KV
  warehouse/
    database.yaml                    # Add backup stanza
    scheduled-backup.yaml            # New: ScheduledBackup CR
    backup-externalsecret.yaml       # New: Azure creds from KV

apps/production/
  linkding/
    backup-cronjob.yaml              # New
    backup-script-configmap.yaml     # New
    backup-config-configmap.yaml     # New
    backup-externalsecret.yaml       # New
    kustomization.yaml               # Update: add backup resources
  audiobookshelf/
    backup-cronjob.yaml              # New
    backup-script-configmap.yaml     # New
    backup-config-configmap.yaml     # New
    backup-externalsecret.yaml       # New
    kustomization.yaml               # Update: add backup resources
```

## Excluded

- **`warehouse-db-dev-cnpg-v1`** — dev data, not worth backing up
- **Audiobookshelf media PVCs** (`audiobooks-pvc`, `podcasts-pvc`) — too large for blob backup, replaceable content. Consider a separate strategy (rsync to NAS) if media becomes irreplaceable.
- **Automated restore for file-based apps** — restore is manual (download archive, extract to PVC). Automating this adds complexity for an infrequent operation. Revisit if rebuilds become common.

## Open Questions

1. **Azure Storage account** — use the existing one backing Actual Budget backups, or create a dedicated one for CNPG?
2. **Auth method for CNPG** — storage key vs SAS token vs managed identity? Storage key is simplest but broadest access.
3. **CNPG bootstrap toggle** — how to manage the `bootstrap.recovery` stanza? Options:
   - Kustomize overlay that's commented out until needed
   - A separate `recovery/` overlay directory only applied during rebuilds
   - Manual `kubectl apply` of a patched Cluster definition during recovery
4. **Backup schedule alignment** — are all backups at 3 AM UTC acceptable, or should they be staggered?
5. **Monitoring/alerting** — should backup failures trigger alerts via kube-prometheus-stack? (Probably yes, but out of scope for this spec.)
