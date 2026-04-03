## ADDED Requirements

### Requirement: Mealie database backups use CloudNativePG native backup features
The Mealie PostgreSQL cluster SHALL use CloudNativePG native Azure Blob object-store backup configuration for continuous archival and scheduled base backups.

#### Scenario: Mealie CNPG cluster is configured for object-store backup
- **WHEN** the Mealie CNPG cluster manifests are rendered
- **THEN** the cluster includes Azure Blob object-store backup configuration
- **AND** the backup configuration uses default Azure credentials through workload identity

#### Scenario: Mealie CNPG cluster has scheduled base backups
- **WHEN** the Mealie database backup resources are applied
- **THEN** the `mealie` namespace contains a `ScheduledBackup` resource for the Mealie CNPG cluster

### Requirement: Mealie filesystem state is backed up to external object storage
The cluster SHALL back up the full Mealie `/app/data` volume to Azure Blob Storage on a schedule independent of the database backup schedule.

#### Scenario: Mealie app data backup job is defined
- **WHEN** the production Mealie backup manifests are rendered
- **THEN** they include a scheduled Kubernetes job that mounts the Mealie `/app/data` volume read-only
- **AND** the job uploads a timestamped archive to Azure Blob Storage

### Requirement: Mealie recovery paths are separated by data type
The operational model SHALL treat CNPG recovery as the primary database recovery path and `/app/data` restore as the filesystem recovery path.

#### Scenario: Recovery resources support database and filesystem restoration separately
- **WHEN** operators review the Mealie deployment and backup resources
- **THEN** the CNPG cluster resources support recovery from CNPG-managed backups
- **AND** the `/app/data` backup resources support restoring application filesystem data without depending on Mealie's built-in export alone

### Requirement: Mealie backup integrations use workload identity
The Mealie database backup flow and `/app/data` archive job SHALL authenticate to Azure Blob Storage through Kubernetes workload identity.

#### Scenario: Azure backup identity is not committed to git
- **WHEN** the Mealie backup manifests are applied
- **THEN** backup jobs and CNPG backup configuration reference the dedicated `mealie-backup` service account and workload identity settings
- **AND** the repository does not contain plaintext Azure backup credentials, storage keys, or backup connection strings

#### Scenario: Workload identity contract matches the Azure handoff
- **WHEN** operators wire the Mealie backup identity into the cluster
- **THEN** the Kubernetes service account subject is `system:serviceaccount:mealie:mealie-backup`
- **AND** the Azure identity contract uses the `id-mealie-backup` managed identity and `fic-mealie-backup` federated credential

### Requirement: Mealie backups use the shared homelab Azure backup account layout
The Mealie backup design SHALL place CNPG and `/app/data` backups in the shared homelab Azure backup storage account located in `rg-jellyhomelab`, using the `mealie` container with `db/` and `files/` prefixes.

#### Scenario: Backup destinations are explicit and shared-account based
- **WHEN** operators review the Mealie backup manifests and runbook notes
- **THEN** the Azure backup destination is documented as the shared homelab backup storage account in `rg-jellyhomelab`
- **AND** the Mealie CNPG and filesystem backups use the `mealie` container with distinct `db/` and `files/` prefixes

#### Scenario: Azure authorization is scoped to the workload container
- **WHEN** operators review the Mealie backup handoff contract
- **THEN** Blob access is granted through `Storage Blob Data Contributor` on the `mealie` container
- **AND** the backup design does not require account-wide storage keys for Mealie backups
