# ActualBudget Backup Identity Terraform Handoff

This document defines the Azure workload identity contract that `jellylabs-dsc` must keep in sync with the ActualBudget backup manifests in `homelab`.

## Goal

Let the `actualbudget-backup` CronJob in `apps/production/actualbudget/` upload backups to Azure Blob Storage without a Kubernetes secret containing storage credentials.

## Required Azure resources

Create or confirm these Azure resources in `rg-jellyhomelab`:

- user-assigned managed identity: `id-actualbudget-backup`
- Blob container: `actualbudget` in storage account `sthomelabbackups`
- role assignment: `Storage Blob Data Contributor` scoped to the `actualbudget` container

## Required federated credential

Create or confirm this federated credential on `id-actualbudget-backup`:

- suggested name: `fic-actualbudget-backup`
- subject: `system:serviceaccount:actualbudget:actualbudget-backup`

## OIDC issuer input

Set `kubernetes_oidc_issuer_url` in `~/git/jellylabs-dsc/infrastructure/azure/homelab/` to the Arc-managed issuer for this cluster before applying Terraform or OpenTofu.

Use the standard subject format:

```text
system:serviceaccount:<namespace>:<service-account>
```

## Outputs consumed by homelab

Keep these outputs available for the Kubernetes manifests and runbook:

- `actualbudget_backup_uami_client_id`
- `actualbudget_backup_uami_principal_id`
- `actualbudget_backup_service_account_subject`
- `actualbudget_backup_container_name`
- `actualbudget_backup_container_resource_id`
- `actualbudget_backup_destination_url`

Expected destination prefix:

- `https://sthomelabbackups.blob.core.windows.net/actualbudget/`

## Runtime contract in homelab

`homelab` will provide:

- `ServiceAccount/actualbudget-backup` annotated with `azure.workload.identity/client-id`
- Pod label `azure.workload.identity/use: "true"`
- CronJob `actualbudget-backup` running `az login --federated-token` before `az storage blob` commands
- ConfigMap values `STORAGE_ACCOUNT_NAME=sthomelabbackups` and `CONTAINER_NAME=actualbudget`

The backup job must not depend on `AZURE_STORAGE_CONNECTION_STRING` or an `ExternalSecret` that renders blob credentials.

## Validation

After the Azure change lands, confirm the Kubernetes subject can authenticate:

1. Trigger a manual run of the `actualbudget-backup` CronJob and confirm the logs show `Logging into Azure with workload identity...` followed by a successful blob upload.
2. Confirm new archives appear under `https://sthomelabbackups.blob.core.windows.net/actualbudget/`.
3. If Azure login fails with `AADSTS700213`, re-check the federated credential subject for `actualbudget-backup`.

## Related docs

- `specs/001-cluster-bootstrap.md`
- `specs/002-data-resilience.md`
