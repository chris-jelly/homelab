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

### Requirement: Mealie file backups use an explicit service account for Azure federation
The system SHALL bind the Mealie file-backup workload to Kubernetes service account `mealie-backup` in namespace `mealie` and SHALL annotate that service account with the Azure client ID for the pre-created Mealie user-assigned managed identity.

#### Scenario: Service account is prepared for Azure federation
- **WHEN** operators review the committed Mealie backup manifests
- **THEN** a `ServiceAccount` named `mealie-backup` exists in namespace `mealie`
- **AND** it carries annotation `azure.workload.identity/client-id`

### Requirement: Mealie file backup pods request Arc workload identity mutation
The system SHALL label each Mealie file-backup pod template with `azure.workload.identity/use: "true"` and SHALL run those consumers with `serviceAccountName: mealie-backup`.

#### Scenario: Backup pod template is eligible for mutation
- **WHEN** a Mealie backup workload pod template is rendered
- **THEN** the pod template includes label `azure.workload.identity/use: "true"`
- **AND** the pod spec sets `serviceAccountName: mealie-backup`

### Requirement: Mealie database backups use workload identity with the CNPG runtime subject
The system SHALL configure Mealie database backups to authenticate to Azure Blob Storage through workload identity, SHALL not require a Mealie-specific Kubernetes secret or ExternalSecret that stores Blob credentials, and SHALL align Azure federation with the actual CNPG service account subject used at runtime.

#### Scenario: Database backup configuration is secretless
- **WHEN** operators inspect the committed Mealie database backup configuration
- **THEN** Azure authentication uses the workload identity path
- **AND** Azure trusts subject `system:serviceaccount:mealie:mealie-db-production-cnpg-v1`
- **AND** no Mealie-specific Blob credential secret is required for that database backup

### Requirement: Mealie file backups use the same Azure workload identity
The system SHALL configure the Mealie file-backup consumer to use the `mealie-backup` service account, SHALL authenticate to Azure Blob through workload identity-aware `azcopy` configuration, and SHALL write backup artifacts to the `files/` prefix inside the `mealie` blob container.

#### Scenario: File backup targets the Mealie files prefix
- **WHEN** operators inspect the committed Mealie file-backup configuration
- **THEN** the workload runs as `mealie-backup`
- **AND** the backup client is configured for workload identity instead of a connection string
- **AND** the Azure Blob destination resolves under `https://sthomelabbackups.blob.core.windows.net/mealie/files/`

### Requirement: Mealie backup rollout includes validation guidance
The system SHALL document how to validate that the Mealie backup consumer was mutated for workload identity and can reach the intended Azure Blob destination.

#### Scenario: Operators can validate the first consumer rollout
- **WHEN** operators follow the Mealie backup rollout guidance
- **THEN** they can verify the service account annotation, pod label, and effective service account on the running backup workload
- **AND** they can confirm that the backup workload can authenticate to the `mealie` blob container without a stored Azure credential secret

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
