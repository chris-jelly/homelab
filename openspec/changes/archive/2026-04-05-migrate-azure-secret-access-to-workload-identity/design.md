## Context

The cluster now has an Arc-managed OIDC issuer and a proven workload identity pattern from the Mealie backup rollout. Despite that platform capability, the primary Azure Key Vault path still depends on a manually created `azure-creds` Kubernetes secret containing a Service Principal client secret, and ActualBudget backups still depend on a storage connection string pulled from Azure Key Vault. The cluster also maintains a second Key Vault (`kv-work-integrations`) and matching `ClusterSecretStore` only to source the Airflow Salesforce connection.

This change crosses multiple layers:

- Kubernetes platform configuration for External Secrets Operator
- application manifests in Airflow and ActualBudget
- bootstrap and runbook documentation in `homelab`
- Azure resource definitions in `~/git/jellylabs-dsc/infrastructure/azure/homelab/`

The design must preserve the current consumer-facing secret contracts where workloads still need secret values, while replacing Azure authentication secrets with workload identity wherever the client supports it.

## Goals / Non-Goals

**Goals:**
- Remove the `azure-creds` bootstrap secret from the External Secrets Operator Azure Key Vault authentication path.
- Standardize homelab Key Vault usage on `kv-jellyhomelabprod`.
- Preserve existing Kubernetes Secret names and key layouts for Airflow connection consumers while changing their upstream Key Vault source.
- Migrate ActualBudget Blob backup authentication from connection-string-based auth to workload identity and Blob RBAC.
- Update bootstrap and operating documentation so the supported Azure integration patterns are internally consistent.
- Retire the `kv-work-integrations` Azure resources after secret consolidation.

**Non-Goals:**
- Introducing per-namespace or per-application Key Vault identities for External Secrets Operator.
- Redesigning all secret flows in the cluster or replacing third-party API keys with non-secret auth.
- Moving in-cluster-generated secrets such as CNPG credentials into Azure Key Vault.
- Changing the Airflow Salesforce connection JSON contract or the Salesforce org setup beyond the vault relocation.
- Reworking ActualBudget backup retention behavior in this change.

## Decisions

### 1. Use one shared workload identity-backed Key Vault store for homelab secrets
The cluster will keep a single shared `ClusterSecretStore`, `azure-kv-store`, pointed at `kv-jellyhomelabprod`. Its authentication method will change from `authType: ServicePrincipal` with `authSecretRef` to a workload identity-compatible Azure Key Vault auth path.

**Rationale:** This removes the long-lived `azure-creds` secret without introducing a phase-3 style access-boundary redesign. It preserves the current operator model where platform-managed `ExternalSecret` objects can continue to reference a single shared store.

**Alternatives considered:**
- **One identity per namespace/app:** better least-privilege boundaries, but out of scope for this change and operationally heavier.
- **Keep the Service Principal secret:** simplest short term, but leaves the cluster with an avoidable bootstrap secret and diverges from the Arc workload identity platform direction.

### 2. Authenticate `azure-kv-store` through a referenced service account
The shared `azure-kv-store` ClusterSecretStore will use `authType: WorkloadIdentity` and a dedicated `serviceAccountRef` named `azure-kv-store-reader` in the `external-secrets` namespace. That referenced service account will be annotated with the client ID of a dedicated Azure user-assigned managed identity. The managed identity will receive the `Key Vault Secrets User` role on `kv-jellyhomelabprod`. The service account name is a documented cross-repo constant, not a Terraform variable.

**Rationale:** The ESO Azure Key Vault docs describe the referenced service account pattern as the usually recommended approach because the controller itself does not need broad Azure permissions. This fits the desired architecture: keep one shared homelab Key Vault store without introducing phase-3 per-namespace complexity, while still avoiding the less desirable mounted-controller-identity pattern.

**Alternatives considered:**
- **Mounted service account on the ESO controller pods:** simpler at first glance, but the docs caution that it broadly grants Azure read capability to anyone able to use the shared store.
- **Attach workload identity to an existing default service account:** less explicit and harder to reason about during debugging.
- **Continue using a generic bootstrap secret only for ESO:** keeps an exception in the platform and defeats the change goal.

### 3. Consolidate Salesforce secret residency into the primary homelab vault
The Salesforce consumer key and private key will be stored in `kv-jellyhomelabprod` under the same secret names already used by the Airflow `ExternalSecret`. The Airflow manifest will switch from `azure-kv-work-integrations-store` to `azure-kv-store`, but the rendered `salesforce-conn` Kubernetes Secret will keep its current name and JSON shape.

**Rationale:** This removes the only known runtime dependency on `kv-work-integrations` while minimizing Airflow-side churn. Keeping secret names stable reduces migration risk and keeps the manifest diff focused on store and vault consolidation rather than data-model changes.

**Alternatives considered:**
- **Keep a second workload-identity-backed store for `kv-work-integrations`:** contradicts the goal of vault consolidation and leaves unnecessary Azure resources behind.
- **Rename the Salesforce secrets during the move:** possible, but increases the number of moving parts and creates avoidable manifest churn.

### 4. Keep the two-lane Azure integration model
The cluster will explicitly use two Azure integration patterns:

- **Key Vault-backed secret synchronization** for workloads that still need a secret value at runtime, via ESO and `azure-kv-store`
- **Direct workload identity + Azure RBAC** for workloads that only need Azure resource access, such as Blob backups

**Rationale:** This matches the actual problem space. Workload identity removes Azure credential secrets, but it does not eliminate application secrets such as OpenAI API keys or Salesforce private keys. Keeping the distinction explicit avoids overgeneralizing workload identity into places where secret values still fundamentally exist.

### 5. Migrate ActualBudget backups by following the Mealie backup identity pattern
ActualBudget backups will switch from `AZURE_STORAGE_CONNECTION_STRING` to a dedicated workload identity. The backup CronJob will use an explicit service account, Azure workload identity pod mutation, and Azure Blob RBAC scoped to a dedicated `actualbudget` container in the existing `sthomelabbackups` storage account. The backup script will authenticate through Azure CLI default credentials rather than a connection string.

**Rationale:** Mealie already established the preferred backup auth pattern in this cluster. Reusing that pattern reduces design novelty, aligns runbooks, and removes another Azure credential secret from the cluster. Reusing `sthomelabbackups` while creating a dedicated `actualbudget` container keeps the Azure layout simple and gives a clean container-level RBAC scope.

**Alternatives considered:**
- **Keep storing a connection string in Key Vault:** preserves current behavior but leaves a secret-based Azure auth path in place.
- **Use a shared backup identity across apps:** simpler Azure inventory, but weakens workload boundaries and couples unrelated backup jobs.
- **Create a separate storage account for ActualBudget:** possible, but unnecessary given the existing shared backup account and clean container-level scoping.

### 6. Retire `kv-work-integrations` only after the homelab consumers are migrated
The Azure cleanup for `kv-work-integrations` and `rg-work-integrations` will happen after the Salesforce secrets have been copied to `kv-jellyhomelabprod`, the Airflow `ExternalSecret` has been repointed, and the new primary-vault-based path is validated.

**Rationale:** This keeps rollback simple during the cutover. The old vault remains available until the new one is proven functional, but it is not part of the target architecture.

## Risks / Trade-offs

- **[ESO workload identity configuration is incomplete]** → Validate the referenced service account annotation, `ClusterSecretStore` `serviceAccountRef`, `authType: WorkloadIdentity` configuration, store readiness, and successful reconciliation of representative `ExternalSecret` resources before removing `azure-creds` from the bootstrap guidance.
- **[Salesforce cutover breaks Airflow connection rendering]** → Keep the Kubernetes Secret name, target key, secret names, and template structure unchanged; only change the upstream store and vault. Validate the resulting `salesforce-conn` secret contents after cutover.
- **[Bootstrap ordering becomes stricter]** → Update `specs/001-cluster-bootstrap.md` to make Arc workload identity readiness an explicit prerequisite before expecting ESO to reconcile Key Vault-backed secrets.
- **[ActualBudget backup image or script does not authenticate cleanly with federated Azure CLI credentials]** → Validate the Azure CLI login flow in a manual backup run before removing the old secret path. If necessary, document the exact CLI auth invocation required by the workload identity-injected environment.
- **[Key Vault consolidation increases dependence on one vault]** → Accept this trade-off for simplicity in this change; mitigation is preserving vault secret naming discipline and keeping the Azure infrastructure defined declaratively in `jellylabs-dsc`.
- **[Terraform cleanup removes `kv-work-integrations` too early]** → Sequence Azure cleanup after homelab manifest cutover and validation, not before.

## Migration Plan

1. Add Azure resources in `jellylabs-dsc` for the External Secrets Operator managed identity, federated credential, and `Key Vault Secrets User` permissions on `kv-jellyhomelabprod`.
2. Move or duplicate the Salesforce secret values into `kv-jellyhomelabprod` using the existing secret names.
3. Update the `azure-kv-store` configuration to use `authType: WorkloadIdentity` with a dedicated referenced service account instead of `azure-creds`.
4. Repoint Airflow Salesforce secret sync to `azure-kv-store` and validate that the rendered `salesforce-conn` secret remains correct.
5. Remove `azure-kv-work-integrations-store` from homelab manifests and docs.
6. Add Azure resources for ActualBudget backup workload identity, the `actualbudget` container in `sthomelabbackups`, and Blob role assignment scoped to that container.
7. Update ActualBudget backup manifests and script to use workload identity-based Blob access and validate with a manual backup run.
8. Update bootstrap and operational docs to remove the `azure-creds` step and all `kv-work-integrations` references.
9. Remove the no-longer-needed app registration / Service Principal path used for `azure-creds`.
10. Remove `kv-work-integrations` and `rg-work-integrations` from `jellylabs-dsc` after all consumers are migrated.

### Rollback
- If ESO workload identity fails, temporarily restore the prior `azure-kv-store` Service Principal configuration and the `azure-creds` bootstrap secret guidance.
- If Salesforce secret cutover fails, repoint the Airflow `ExternalSecret` back to the old store until the primary vault contents are corrected.
- If ActualBudget backup workload identity fails, revert its backup job to the previous secret-based connection string path until the federated identity or Blob RBAC is fixed.
- Delay Azure resource deletion for `kv-work-integrations` until the Kubernetes-side migration is stable.

## Resolved Design Inputs

- ESO Key Vault reads will use Azure RBAC through the `Key Vault Secrets User` role on `kv-jellyhomelabprod`.
- Azure-side cleanup will remove the now-unused app registration / Service Principal path that backed the `azure-creds` secret.
- ActualBudget does not yet have a backup target in `rg-jellyhomelab`, so this change will create a dedicated `actualbudget` container in the existing `sthomelabbackups` account and scope Blob RBAC there.
- The preferred ESO workload identity pattern is the documented referenced service account approach (`authType: WorkloadIdentity` plus `serviceAccountRef`), using the fixed service account name `azure-kv-store-reader`, unless implementation proves the installed chart version lacks support for the required fields.

## Open Questions

- Confirm during implementation that the installed ESO chart/CRD version supports the referenced service account workload identity fields exactly as documented in the current Azure Key Vault provider docs.