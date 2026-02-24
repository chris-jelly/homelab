## Why

The `warehouse-postgres-conn` ExternalSecret currently sources the database password from Azure Key Vault (`app--warehouse-postgres-password--prod`), but this KV secret does not exist and was never provisioned. The CNPG operator already generates a complete connection URI in the `warehouse-db-production-cnpg-v1-app` secret within the `warehouse` namespace. Manually copying credentials into Azure KV creates a bootstrap dependency (fresh deploys fail until someone runs `az keyvault secret set`), a drift risk (CNPG password rotation invalidates the KV copy), and an unnecessary external dependency for what is an in-cluster secret. The Airflow metadata database already uses a cross-namespace ESO Kubernetes provider to read its CNPG secret directly -- the warehouse connection should use the same pattern.

## What Changes

- **Add** cross-namespace RBAC (Role + RoleBinding in `warehouse` namespace) granting the existing `external-secrets-kubernetes-reader` ServiceAccount in the `airflow` namespace read access to secrets in the `warehouse` namespace
- **Add** a `kubernetes-warehouse` SecretStore in the `airflow` namespace pointing at `remoteNamespace: warehouse`
- **Modify** the `warehouse-postgres-conn` ExternalSecret to source the `uri` field directly from the CNPG-generated secret (`warehouse-db-production-cnpg-v1-app`) via the new Kubernetes SecretStore, instead of composing a URI from Azure KV
- **Remove** the Azure KV dependency (`app--warehouse-postgres-password--prod`) from the warehouse connection provisioning path

## Capabilities

### New Capabilities
- `warehouse-secret-access`: Cross-namespace RBAC and ESO SecretStore allowing the `airflow` namespace to read CNPG-generated secrets from the `warehouse` namespace. This pattern is reusable by future consumers (e.g., streamlit) that need warehouse database credentials.

### Modified Capabilities
- `airflow-dag-connections`: The warehouse PostgreSQL connection ExternalSecret changes its data source from Azure KV (`azure-kv-store` ClusterSecretStore) to the in-cluster CNPG secret (`kubernetes-warehouse` SecretStore). The resulting K8s Secret name and key (`warehouse-postgres-conn` / `AIRFLOW_CONN_WAREHOUSE_POSTGRES`) remain unchanged.

## Impact

- **ExternalSecrets**: `warehouse-postgres-conn` ExternalSecret manifest is rewritten to use the Kubernetes provider instead of Azure KV. No change to the output secret name or key.
- **RBAC**: New Role and RoleBinding in the `warehouse` namespace. The existing `external-secrets-kubernetes-reader` ServiceAccount in `airflow` is reused -- no new SA needed.
- **Azure Key Vault**: The `app--warehouse-postgres-password--prod` secret in `kv-jellyhomelabprod` is no longer referenced. It can be deleted from KV at any time (it never existed anyway).
- **Kustomization**: `apps/production/airflow/kustomization.yaml` may need to reference a new manifest file for the warehouse RBAC/SecretStore resources, or they can be added to the existing `database-externalsecret.yaml` multi-document file.
- **Bootstrap**: Fresh cluster deploys no longer require manual Azure KV secret provisioning for the warehouse password. CNPG creates the secret, ESO reads it, Airflow gets the connection -- fully automated.
- **Future consumers**: The `warehouse-secret-access` RBAC pattern can be extended with additional RoleBindings for other namespaces (e.g., streamlit) that need warehouse access.
