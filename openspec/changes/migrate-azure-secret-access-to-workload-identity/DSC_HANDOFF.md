# DSC Handoff

Change: `migrate-azure-secret-access-to-workload-identity`
Repo: `~/git/jellylabs-dsc`
Stack: `infrastructure/azure/homelab`

## Goal

Make the Azure-side changes needed for homelab to:

1. move External Secrets Operator Key Vault access to workload identity
2. consolidate Salesforce secrets into `kv-jellyhomelabprod`
3. add an ActualBudget Blob backup target and identity
4. remove the old `azure-creds` app registration / Service Principal path after cutover
5. remove `kv-work-integrations` and `rg-work-integrations` after validation

## Expected Kubernetes subjects

### External Secrets Operator referenced service account

This is a fixed documented constant, not a Terraform variable.

```text
system:serviceaccount:external-secrets:azure-kv-store-reader
```

### ActualBudget backup service account

```text
system:serviceaccount:actualbudget:actualbudget-backup
```

## Required Azure RBAC

### ESO Key Vault identity
- Role: `Key Vault Secrets User`
- Scope: `kv-jellyhomelabprod`

### ActualBudget backup identity
- Role: `Storage Blob Data Contributor`
- Scope: container `actualbudget` in storage account `sthomelabbackups`

## Azure resources to add or update

### 1. ESO workload identity
- user-assigned managed identity for ESO Key Vault reads
- federated credential for subject `system:serviceaccount:external-secrets:azure-kv-store-reader`
- Key Vault role assignment on `kv-jellyhomelabprod`

### 2. ActualBudget backup workload identity
- user-assigned managed identity for ActualBudget backups
- federated credential for `system:serviceaccount:actualbudget:actualbudget-backup`
- storage container `actualbudget` in `sthomelabbackups`
- Blob role assignment scoped to that container

### 3. Salesforce secret residency
Copy these secrets into `kv-jellyhomelabprod` and keep the names unchanged:

- `app--salesforce-consumer-key--prod`
- `app--salesforce-private-key--prod`

## Azure resources to remove after homelab validation

- the legacy app registration / Service Principal path used by `azure-creds`
- `kv-work-integrations`
- `rg-work-integrations`

## Suggested outputs

Expose these outputs if they do not already exist:

- `eso_kv_uami_client_id`
- `eso_kv_uami_principal_id`
- `actualbudget_backup_uami_client_id`
- `actualbudget_backup_uami_principal_id`
- `actualbudget_backup_container_name`
- `actualbudget_backup_container_resource_id`
- `actualbudget_backup_destination_url` (optional)

## Suggested implementation checklist

- [ ] Add ESO UAMI and federated credential
- [ ] Grant ESO UAMI `Key Vault Secrets User` on `kv-jellyhomelabprod`
- [ ] Add ActualBudget UAMI and federated credential
- [ ] Create container `actualbudget` in `sthomelabbackups`
- [ ] Grant ActualBudget UAMI `Storage Blob Data Contributor` on that container
- [ ] Copy Salesforce secrets into `kv-jellyhomelabprod`
- [ ] Apply and capture outputs needed by homelab
- [ ] Wait for homelab-side cutover and validation
- [ ] Remove the legacy `azure-creds` app registration / Service Principal path
- [ ] Remove `kv-work-integrations` and `rg-work-integrations`

## Sequencing notes

Keep `kv-work-integrations` in place until homelab confirms all of the following:

- `azure-kv-store` works with `WorkloadIdentity` and `serviceAccountRef`
- Airflow `salesforce-conn` renders correctly from `kv-jellyhomelabprod`
- an ActualBudget manual backup run succeeds

## Notes

- Reuse the existing Arc OIDC issuer already wired into `homelab.auto.tfvars`.
- Keep Salesforce secret names unchanged to minimize homelab manifest churn.
- Prefer container-level RBAC for ActualBudget instead of broader storage-account scope.
- Treat `azure-kv-store-reader` as a cross-repo constant owned by homelab, not a Terraform variable.
- This handoff is for a human or agent to read, not a script to run.
