## ADDED Requirements

### Requirement: Primary homelab Key Vault access uses workload identity
The system SHALL authenticate the shared `azure-kv-store` ClusterSecretStore to `kv-jellyhomelabprod` through Azure workload identity rather than a Kubernetes secret containing Service Principal credentials.

#### Scenario: ClusterSecretStore targets the primary homelab vault
- **WHEN** operators review the `azure-kv-store` ClusterSecretStore configuration
- **THEN** it targets `https://kv-jellyhomelabprod.vault.azure.net/`
- **AND** it does not reference `authSecretRef.clientSecret`

#### Scenario: ClusterSecretStore uses a referenced workload identity service account
- **WHEN** operators review the External Secrets Operator Azure Key Vault authentication configuration
- **THEN** the `azure-kv-store` ClusterSecretStore uses `authType: WorkloadIdentity`
- **AND** it references `ServiceAccount/azure-kv-store-reader` in the `external-secrets` namespace via `serviceAccountRef`
- **AND** that service account is annotated with the Azure managed identity client ID

### Requirement: External Secrets bootstrap no longer depends on azure-creds
The system SHALL document and support a cluster bootstrap path where External Secrets Operator can reconcile Key Vault-backed secrets without creating the `azure-creds` secret in the `external-secrets` namespace.

#### Scenario: Bootstrap runbook omits azure-creds creation for ESO
- **WHEN** operators follow the cluster bootstrap runbook after Arc workload identity is enabled
- **THEN** the documented External Secrets Operator bootstrap path does not require `kubectl create secret generic azure-creds`

### Requirement: Homelab uses a single Azure Key Vault store for ESO-backed secrets
The system SHALL use `azure-kv-store` as the shared Key Vault-backed ClusterSecretStore for homelab workloads and SHALL not require `azure-kv-work-integrations-store`.

#### Scenario: Secondary work-integrations store is absent
- **WHEN** operators review the homelab manifests for Key Vault-backed stores
- **THEN** `azure-kv-work-integrations-store` is not defined
- **AND** homelab `ExternalSecret` resources that read from Azure Key Vault reference `azure-kv-store`