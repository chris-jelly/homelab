## Requirements

### Requirement: Mealie file-backup logic stays small and local
The system SHALL keep the Mealie file-backup implementation small enough that it can remain as a local Kubernetes-mounted shell script, limited to archive creation and upload.

#### Scenario: Kubernetes manifests do not host backup logic
- **WHEN** operators inspect the committed Mealie file-backup manifests
- **THEN** the CronJob may mount a small ConfigMap-hosted shell script for backup behavior
- **AND** that script is limited to archive creation and upload
- **AND** the Kubernetes manifests do not embed retention pruning logic

### Requirement: The Mealie backup runtime contract stays explicit
The system SHALL define a stable runtime contract for the Mealie file-backup job, including the required environment variables, workload identity expectations, source path, and destination path.

#### Scenario: Runtime contract is documented for the image repo
- **WHEN** operators inspect the Mealie backup handoff documentation in `homelab`
- **THEN** they can identify the env vars and filesystem contract the helper image must support
- **AND** they can see that workload identity-based AzCopy authentication remains part of that contract

### Requirement: File-backup retention is enforced by Azure storage lifecycle policy
The system SHALL enforce Mealie file-backup retention for the `mealie/files/` prefix through Azure Blob lifecycle management rather than in-job prune logic.

#### Scenario: Retention is described as storage policy
- **WHEN** operators inspect the Mealie backup handoff documentation in `homelab`
- **THEN** they can identify that retention for `mealie/files/` is owned by Azure storage lifecycle management
- **AND** they can see the intended retention window that Terraform must enforce

### Requirement: The Mealie CronJob remains declarative after the refactor
The system SHALL keep the Mealie CronJob focused on deployment wiring, including schedule, service account, labels, mounts, image reference, and only the configuration needed for the simplified backup script.

#### Scenario: CronJob declares wiring, not implementation
- **WHEN** operators inspect the committed Mealie CronJob manifest after the refactor
- **THEN** the manifest declares the backup image, service account, and required configuration inputs
- **AND** it does not include retention-pruning behavior in the CronJob path
