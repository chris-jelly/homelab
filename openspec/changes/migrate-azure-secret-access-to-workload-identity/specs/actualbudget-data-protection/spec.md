## ADDED Requirements

### Requirement: ActualBudget backup uploads use workload identity
The ActualBudget backup job SHALL authenticate to Azure Blob Storage through Kubernetes workload identity rather than a storage connection string secret.

#### Scenario: Backup job uses workload identity contract
- **WHEN** operators review the ActualBudget backup CronJob and related service account resources
- **THEN** the backup job uses a dedicated Kubernetes service account annotated with the Azure managed identity client ID
- **AND** the backup pod template includes the `azure.workload.identity/use: "true"` label

#### Scenario: Backup job no longer depends on connection string secret
- **WHEN** operators review the ActualBudget backup manifests
- **THEN** the CronJob does not import `AZURE_STORAGE_CONNECTION_STRING` from a Kubernetes Secret
- **AND** the namespace does not require an `ExternalSecret` that produces storage credentials for the backup job

### Requirement: ActualBudget backup destination remains Azure Blob Storage
The ActualBudget backup flow SHALL continue to upload compressed backups from the `/data` volume to Azure Blob Storage using the configured backup container and prefix.

#### Scenario: Backup upload contract is preserved
- **WHEN** the ActualBudget backup job runs successfully
- **THEN** it uploads a backup archive to the configured Azure Blob container using the existing backup naming convention
- **AND** the backup still originates from the mounted `/data` volume

### Requirement: ActualBudget backup Azure access is scoped to the backup target
The Azure identity used by the ActualBudget backup job SHALL receive Blob data permissions scoped to a dedicated ActualBudget backup container in the shared homelab backup storage account rather than broad account-wide credentials embedded in a secret.

#### Scenario: Backup identity uses RBAC on the destination
- **WHEN** operators review the Azure identity contract for ActualBudget backups
- **THEN** it defines a federated credential for the ActualBudget backup service account subject
- **AND** Blob access is granted through Azure RBAC on the dedicated ActualBudget backup container in `sthomelabbackups`