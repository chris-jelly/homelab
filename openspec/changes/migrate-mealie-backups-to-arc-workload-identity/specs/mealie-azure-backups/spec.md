## ADDED Requirements

### Requirement: Mealie backup workloads use the shared Azure workload identity contract
The system SHALL bind Mealie backup workloads to Kubernetes service account `mealie-backup` in namespace `mealie` and SHALL annotate that service account with the Azure client ID for the pre-created Mealie user-assigned managed identity.

#### Scenario: Service account is prepared for Azure federation
- **WHEN** operators review the committed Mealie backup manifests
- **THEN** a `ServiceAccount` named `mealie-backup` exists in namespace `mealie`
- **AND** it carries annotation `azure.workload.identity/client-id`

### Requirement: Mealie backup pods request Arc workload identity mutation
The system SHALL label each Mealie backup consumer pod template with `azure.workload.identity/use: "true"` and SHALL run those consumers with `serviceAccountName: mealie-backup`.

#### Scenario: Backup pod template is eligible for mutation
- **WHEN** a Mealie backup workload pod template is rendered
- **THEN** the pod template includes label `azure.workload.identity/use: "true"`
- **AND** the pod spec sets `serviceAccountName: mealie-backup`

### Requirement: Mealie database backups use workload identity instead of Blob secrets
The system SHALL configure Mealie database backups to authenticate to Azure Blob Storage through workload identity and SHALL not require a Mealie-specific Kubernetes secret or ExternalSecret that stores Blob credentials.

#### Scenario: Database backup configuration is secretless
- **WHEN** operators inspect the committed Mealie database backup configuration
- **THEN** Azure authentication uses the workload identity path
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
