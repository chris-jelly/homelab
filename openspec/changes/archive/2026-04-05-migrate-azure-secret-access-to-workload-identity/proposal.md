## Why

Azure Arc workload identity is now available in the cluster and Mealie backups already use it successfully, but External Secrets Operator still depends on the manually created `azure-creds` Service Principal secret and ActualBudget backups still depend on a storage connection string secret. The cluster also still carries a second Key Vault (`kv-work-integrations`) only to source the Airflow Salesforce connection, which adds avoidable infrastructure and documentation complexity.

## What Changes

- Migrate External Secrets Operator Azure Key Vault access from the `azure-creds` bootstrap secret to Azure workload identity.
- Consolidate homelab Key Vault usage onto `kv-jellyhomelabprod` and remove the `azure-kv-work-integrations-store` ClusterSecretStore from the cluster.
- Move the Airflow Salesforce connection source-of-truth from `kv-work-integrations` to `kv-jellyhomelabprod` without changing the rendered Kubernetes Secret contract consumed by Airflow.
- Convert ActualBudget backup authentication from an Azure storage connection string secret to workload identity with Blob RBAC, following the Mealie backup pattern.
- Update bootstrap and operational documentation so new cluster bring-up no longer requires creating `azure-creds` for External Secrets Operator and no longer references `kv-work-integrations`.
- Retire the Azure infrastructure for `kv-work-integrations` and its resource group after secrets are consolidated into the primary homelab vault.

## Capabilities

### New Capabilities
- `azure-key-vault-workload-identity`: Authenticate External Secrets Operator to the primary homelab Azure Key Vault through Azure workload identity instead of a bootstrap Service Principal secret.
- `actualbudget-data-protection`: Authenticate ActualBudget backup uploads to Azure Blob Storage through workload identity instead of a storage connection string secret.

### Modified Capabilities
- `airflow-dag-connections`: Change the Salesforce connection secret source from `azure-kv-work-integrations-store` / `kv-work-integrations` to `azure-kv-store` / `kv-jellyhomelabprod` while preserving the Airflow-facing secret shape.
- `azure-arc-workload-identity`: Extend the documented standard Azure workload identity pattern to cover External Secrets Operator as a platform consumer and remove the remaining `azure-creds` bootstrap dependency from the cluster bootstrap path.

## Impact

- Affected Kubernetes code: `infrastructure/configs/base/external-secrets/`, `infrastructure/controllers/base/external-secrets/`, `apps/production/airflow/`, `apps/production/actualbudget/`, and bootstrap/runbook docs.
- Affected Azure infrastructure: `~/git/jellylabs-dsc/infrastructure/azure/homelab/` for the ESO managed identity and federated credential, ActualBudget backup identity and Blob role assignment, Salesforce secret residency in the primary vault, and retirement of `kv-work-integrations` / `rg-work-integrations`.
- Affected systems: Azure Arc workload identity, External Secrets Operator, Azure Key Vault, Azure Blob Storage, Airflow Salesforce integration, ActualBudget backups, and cluster bootstrap operations.
- Operational impact: simplifies bootstrap, removes one long-lived Azure client secret from the cluster, reduces Key Vault sprawl, and standardizes Azure-integrated backup workloads on federated auth.