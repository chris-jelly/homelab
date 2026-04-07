## 1. Azure workload identity foundations

- [x] 1.1 [dsc] Add `jellylabs-dsc` resources for an External Secrets Operator managed identity, federated credential, and `Key Vault Secrets User` permissions on `kv-jellyhomelabprod` in `infrastructure/azure/homelab/`.
- [x] 1.2 [dsc] Add `jellylabs-dsc` resources for an ActualBudget backup managed identity, federated credential, a dedicated `actualbudget` container in `sthomelabbackups`, and Blob RBAC scoped to that container.
- [x] 1.3 [dsc] Copy or move the Salesforce secret values into `kv-jellyhomelabprod` using the existing secret names consumed by homelab.

## 2. External Secrets Operator and Key Vault consolidation

- [x] 2.1 [homelab] Add the dedicated referenced service account `azure-kv-store-reader` for `azure-kv-store` in the `external-secrets` namespace and annotate it with the ESO managed identity client ID.
- [x] 2.2 [homelab] Update `infrastructure/configs/base/external-secrets/cluster-secret-store.yaml` so `azure-kv-store` uses `authType: WorkloadIdentity` with `serviceAccountRef` against `kv-jellyhomelabprod`, and remove `azure-kv-work-integrations-store`.
- [x] 2.3 [homelab] Update Key Vault-backed `ExternalSecret` consumers as needed so all homelab Azure Key Vault reads use `azure-kv-store`.

## 3. Workload migrations

- [x] 3.1 [homelab] Update `apps/production/airflow/salesforce-conn-externalsecret.yaml` to source Salesforce secrets from `azure-kv-store` without changing the rendered `salesforce-conn` secret contract.
- [x] 3.2 [homelab] Migrate ActualBudget backup manifests to workload identity by adding a dedicated service account, workload identity pod label, and removal of the backup credential `ExternalSecret`.
- [x] 3.3 [homelab] Update the ActualBudget backup script and related config so Blob uploads authenticate with workload identity instead of `AZURE_STORAGE_CONNECTION_STRING`.

## 4. Documentation, validation, and cleanup

- [x] 4.1 [homelab] Update `specs/001-cluster-bootstrap.md` to remove the `azure-creds` bootstrap step and document the referenced-service-account ESO workload identity pattern using the fixed service account name `azure-kv-store-reader`.
- [x] 4.2 [homelab] Update runbooks and handoff docs to remove `kv-work-integrations` references and document the new ActualBudget backup auth path.
- [x] 4.3 [both] Validate representative Key Vault-backed `ExternalSecret` reconciliations, the Airflow Salesforce connection secret, and a manual ActualBudget backup run after the migration.
- [x] 4.4 [dsc] Remove the no-longer-needed app registration / Service Principal path for `azure-creds`, then remove `kv-work-integrations` and `rg-work-integrations` from `jellylabs-dsc` after the new primary-vault and workload identity paths are confirmed healthy.
