## Why

The cluster does not yet have a recipe management application, and Mealie is a good fit for a small household service that can share the cluster's existing GitOps, CNPG, and backup patterns. This is a good time to add it because the repo already has a clear backup direction, and Mealie gives us a practical chance to exercise CloudNativePG backup and recovery for a real application.

## What Changes

- Add a production Mealie deployment managed with plain Kubernetes manifests and Flux instead of a third-party Helm chart.
- Add a Mealie PostgreSQL cluster managed by CloudNativePG in the `mealie` namespace.
- Add native CNPG backup configuration and scheduled base backups for the Mealie database using Azure Blob Storage.
- Add file-based backups for Mealie's `/app/data` volume to Azure Blob Storage using the same CronJob shape, but with workload identity instead of storage secrets.
- Add OpenAI integration configuration for Mealie, with the API key sourced from Azure Key Vault through External Secrets.
- Add the required ingress, certificate, Homepage discovery metadata, secrets, and Kustomize wiring so Mealie deploys cleanly through the existing production cluster layout.
- Document the operational boundaries between primary database recovery through CNPG and secondary file restore for Mealie application data.
- Make the Azure backup placement explicit by consuming the Mealie backup handoff contract: shared account `sthomelabbackups` in `rg-jellyhomelab`, `mealie` container, `db/` and `files/` prefixes, and the `id-mealie-backup` workload identity binding.

## Capabilities

### New Capabilities
- `mealie-deployment`: Deploy Mealie as a production application in the homelab with plain manifests, persistent app storage, ingress, Homepage discoverability, optional OpenAI integration, and externalized configuration.
- `mealie-data-protection`: Protect Mealie data with CNPG native PostgreSQL backups and scheduled file backups for `/app/data` stored in Azure Blob Storage under the shared homelab backup account layout.

### Modified Capabilities
- None.

## Impact

- Affected code: `apps/base/`, `apps/production/`, `databases/data/`, and cluster Kustomizations that include those resources.
- Affected systems: Flux CD, Traefik ingress, cert-manager, External Secrets Operator, Azure Key Vault, Azure Blob Storage, and CloudNativePG.
- Operational impact: introduces a new stateful application, a new CNPG cluster, a new OpenAI secret in Azure Key Vault, workload identity wiring for backup consumers through `id-mealie-backup`, a shared `mealie` backup container with `db/` and `files/` prefixes in Azure Blob Storage, and a new recovery path to validate.
