## 1. Application manifests

- [x] 1.1 Create `apps/base/mealie/` with the namespace, deployment, service, persistent volume claim, and base kustomization resources.
- [x] 1.2 Configure the Mealie deployment with pinned image version, persistent `/app/data` storage, health probes, resource requests and limits, secure pod settings, and PostgreSQL environment wiring.
- [x] 1.3 Create `apps/production/mealie/` with the production kustomization, ingress, certificate, Homepage annotations, and app-facing ExternalSecret resources.
- [x] 1.4 Wire Mealie application secrets for database access and OpenAI integration from External Secrets or generated cluster secrets.
- [x] 1.5 Add Mealie to Homepage discovery through `gethomepage.dev/*` ingress annotations and confirm no separate curated tile is needed.
- [x] 1.6 Add Mealie to `apps/production/kustomization.yaml` so Flux includes the production app manifests.

## 2. Database and backup resources

- [x] 2.1 Create `databases/data/mealie/` with a CNPG `Cluster` manifest in the `mealie` namespace and a matching kustomization.
- [x] 2.2 Configure the Mealie CNPG cluster for PostgreSQL 15, app-private access, and Azure Blob native backup settings.
- [x] 2.3 Add a `ScheduledBackup` resource and workload identity-backed backup configuration for the Mealie CNPG cluster.
- [x] 2.4 Create Mealie `/app/data` backup resources in `apps/production/mealie/`, including the backup config ConfigMap, backup script ConfigMap, and backup CronJob.
- [x] 2.5 Add the Mealie database resources to `databases/data/kustomization.yaml` so Flux includes the CNPG cluster.
- [ ] 2.6 Add Kubernetes workload identity wiring for Mealie backups: create the `mealie-backup` ServiceAccount, annotate it with `mealie_backup_uami_client_id`, add required pod labels for Azure workload identity injection, and bind the CNPG and `/app/data` backup workloads to that service account as appropriate.

## 3. External systems and configuration

- [ ] 3.1 Create or confirm the shared Azure backup storage account in `rg-jellyhomelab`, the `mealie` container, and the `db/` and `files/` prefixes using the published OpenTofu outputs.
- [ ] 3.2 Create or confirm the Azure workload identity contract for Mealie backups: `id-mealie-backup`, `fic-mealie-backup`, the `system:serviceaccount:mealie:mealie-backup` federated subject, and the `Storage Blob Data Contributor` role assignment on the `mealie` container.
- [ ] 3.3 Add the required Azure Key Vault secrets for Mealie, keeping `secret/mealie-openai-api-key` and any other non-backup secret material still required by the deployment.
- [ ] 3.4 Create or confirm the production DNS record for the Mealie hostname.

## 4. Validation and recovery

- [ ] 4.1 Validate that Flux renders and reconciles the new Mealie app and CNPG resources successfully.
- [ ] 4.2 Verify that Mealie is reachable through ingress, is discoverable in Homepage, uses the CNPG database, and writes persistent data under `/app/data`.
- [ ] 4.3 Verify that the OpenAI integration is configured from secret material without committing the API key to git.
- [ ] 4.4 Verify that CNPG scheduled backups and Mealie `/app/data` backup jobs execute successfully against the shared Azure Blob backup account without Kubernetes backup credential secrets.
- [ ] 4.5 Run and document a CNPG recovery drill for the Mealie database.
- [ ] 4.6 Run and document a separate `/app/data` restore drill for Mealie filesystem state.
