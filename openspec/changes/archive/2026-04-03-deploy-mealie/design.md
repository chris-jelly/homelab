## Context

This cluster uses Flux CD and Kustomize for application delivery, External Secrets Operator with Azure Key Vault for secret material, and CloudNativePG for PostgreSQL workloads. The current application layer mixes plain manifests and HelmReleases, but the Mealie exploration concluded that a plain-manifest deployment is a better fit than adopting a stale third-party chart.

Mealie needs both a PostgreSQL database and persistent filesystem storage under `/app/data`. The repo already has a desired backup model in `specs/002-data-resilience.md`: CNPG native object-store backups for PostgreSQL workloads and CronJob-based archival for file-backed application state. Mealie is also a good candidate to validate CNPG backup and recovery in practice, since the database is single-tenant and low-risk compared with shared data systems.

The cluster standards document currently says databases should use dedicated namespaces, but current CNPG usage already places app-private clusters in app namespaces. For Mealie, the database is only used by Mealie, so the operational model should optimize for simplicity over strict separation.

## Goals / Non-Goals

**Goals:**
- Deploy Mealie with plain manifests that fit the repo's existing Flux and Kustomize layout.
- Run the Mealie CNPG cluster in the `mealie` namespace as an app-private dependency.
- Persist Mealie filesystem state on a PVC mounted at `/app/data`.
- Back up the Mealie database with CNPG native Azure Blob backup features, including scheduled base backups.
- Back up `/app/data` to Azure Blob with the same CronJob shape used elsewhere in the repo, but authenticate with workload identity instead of storage secrets.
- Keep secrets out of git and source non-backup secrets through External Secrets.
- Enable Mealie's OpenAI integration with the API key sourced from Azure Key Vault.
- Make Mealie discoverable in Homepage as part of the initial app rollout.
- Make the recovery model explicit: CNPG recovery is the primary database recovery path, and `/app/data` restore is the secondary file recovery path.

**Non-Goals:**
- Building or maintaining a custom Helm chart for Mealie.
- Implementing advanced Mealie features such as SMTP or OIDC during the initial deployment.
- Leaving the Mealie database role permanently elevated for application-driven restores.
- General cleanup of existing database namespace conventions outside the Mealie scope.

## Decisions

### Deploy Mealie with plain Kubernetes manifests
Mealie will be deployed with a `Deployment`, `Service`, `PersistentVolumeClaim`, and production overlay resources instead of a third-party Helm chart.

Rationale:
- The explored chart is behind upstream Mealie versions and has weak defaults for persistence and security.
- The app is simple enough that plain manifests are easy to own.
- Plain manifests make it easier to align probes, security context, secret wiring, and backup resources with repo conventions.

Alternatives considered:
- Use the third-party Helm chart with extensive overrides: faster at first, but dependent on a stale chart and chart-specific constraints.
- Fork or vendor the chart: offers control, but creates a chart maintenance burden without clear benefit for a single app.

### Keep the Mealie CNPG cluster in the `mealie` namespace
The database cluster will live in the same namespace as the application.

Rationale:
- Mealie is the only consumer of the database.
- Same-namespace deployment simplifies secret wiring, recovery operations, and incident response.
- This matches the repo's current practical pattern for app-private CNPG clusters.

Alternatives considered:
- Dedicated `mealie-db` namespace: cleaner separation on paper, but more cross-namespace secret and operational overhead for little gain.

### Use CNPG native backup and recovery as the primary database recovery path
The CNPG `Cluster` will include Azure Blob object-store backup configuration, target the latest supported CloudNativePG and Barman plugin versions used in the repo, set `azureCredentials.useDefaultAzureCredentials: true`, and rely on a `ScheduledBackup` resource for regular base backups.

Rationale:
- This matches the cluster's stated data resilience strategy.
- CNPG recovery is the correct disaster-recovery mechanism for a PostgreSQL-backed application.
- Mealie provides a safe, useful first real workload for practicing CNPG recovery.
- The Azure handoff package already defines workload identity and container-scoped Blob access for this backup flow.

Alternatives considered:
- Rely on Mealie's built-in backup UI for primary recovery: insufficient, because it is an application-level export and Postgres restores require temporary superuser access.

### Back up `/app/data` with the same CronJob shape used by Actual Budget
The production overlay will include a CronJob, script ConfigMap, config ConfigMap, and workload identity wiring to archive the `/app/data` PVC to Azure Blob Storage.

Rationale:
- Mealie stores built-in backup ZIP files and other filesystem state under `/app/data`.
- File backup complements, but does not replace, CNPG recovery.
- The repo already has a working CronJob shape to copy and adapt without reusing storage-secret handling.

Alternatives considered:
- Skip file backup and trust only the database: risks losing uploads and other local state.
- Back up only `/app/data/backups`: simpler, but incomplete for full app restore.

### Use a shared Azure backup storage account in `rg-jellyhomelab`
Mealie backups will live in the existing homelab Azure resource group, `rg-jellyhomelab`, and use the shared storage account `sthomelabbackups` rather than a Mealie-specific account. Isolation should happen at the workload container boundary, with both backup flows writing to the `mealie` container under the `db/` and `files/` prefixes.

Rationale:
- The homelab backup pattern is small-scale and operationally simple, so a storage account per workload adds management overhead without much security or resiliency benefit.
- A shared account with one container per workload keeps naming and lifecycle management simple.
- Workload-level separation is still preserved by reserving the `mealie` container for this app and splitting database and filesystem artifacts by prefix.

Alternatives considered:
- Create a dedicated storage account for Mealie backups: offers stronger account-level isolation, but increases Azure resource sprawl for a low-scale homelab pattern.

### Use one Kubernetes service account for Mealie backup consumers
Mealie backup consumers will use a dedicated Kubernetes `ServiceAccount` named `mealie-backup`, mapped to the Azure user-assigned managed identity `id-mealie-backup` through the `fic-mealie-backup` federated identity credential.

Rationale:
- One service account gives both CNPG backup pods and the filesystem backup job a single Azure identity boundary to manage.
- A dedicated service account avoids the default namespace service account and makes workload identity wiring explicit.
- The binding is concrete: `system:serviceaccount:mealie:mealie-backup`.

Alternatives considered:
- Separate service accounts for CNPG and file backups: tighter separation, but more identity wiring for little benefit in this homelab scope.

### Authenticate Azure backup access with workload identity
Mealie backup consumers will authenticate to Azure Blob Storage through workload identity and the `id-mealie-backup` user-assigned managed identity (UAMI), not Key Vault-synced connection strings or storage keys.

Rationale:
- This removes long-lived backup credentials from Kubernetes Secrets and Azure Key Vault.
- The model matches the desired Azure posture for cluster workloads that access shared storage.
- Workload identity still works with the shared backup account layout and dedicated backup service account.
- Azure authorization is scoped through the `Storage Blob Data Contributor` role assignment on the `mealie` container.

Alternatives considered:
- Store connection strings or storage keys in Key Vault and sync them into Kubernetes: simpler to wire at first, but weaker than federated identity and adds secret rotation overhead.

### Enable OpenAI integration through External Secrets
The initial deployment should wire Mealie's OpenAI integration, with the API key delivered from Azure Key Vault through an app-facing `ExternalSecret`.

Rationale:
- This keeps the deployment aligned with the cluster's secret-management standard.
- OpenAI-backed recipe features are user-visible enough to justify inclusion in the first Mealie rollout.
- The integration is low-risk when the key is externalized and optional app behavior is otherwise unchanged.

Alternatives considered:
- Leave OpenAI for a follow-up change: reduces scope slightly, but delays a feature already known to be wanted and does not materially complicate the manifest design.

### Include Homepage discovery metadata from day one
The Mealie ingress should carry `gethomepage.dev/*` annotations so the service appears in Homepage without a separate curated entry.

Rationale:
- Repo standards require an explicit Homepage presence decision for Homepage-worthy apps.
- Mealie is a user-facing web app and should be easy to find after deployment.
- Ingress annotations keep ownership local to the app definition.

Alternatives considered:
- Add a manual Homepage tile later: easier to forget and more likely to drift from the deployed ingress.

### Keep initial app configuration focused but complete for day-one usability
The initial deployment will configure the values needed for a secure and discoverable deployment: base URL, timezone, database connection, signup policy, ingress, TLS, persistent storage, Homepage metadata, and OpenAI secret wiring.

Rationale:
- Keeps the first deployment focused and easier to validate.
- Avoids expanding scope into email or SSO before the core service is healthy.

Alternatives considered:
- Enable SMTP or OIDC immediately: useful later, but not required to land the base service.

## Risks / Trade-offs

- [Third-party image or app changes require manual manifest updates] -> Pin a specific Mealie image tag and treat upgrades as explicit follow-up changes.
- [CNPG recovery is designed but untested in this cluster] -> Include a post-deploy recovery exercise and document the exact recovery procedure.
- [File backups can drift from actual restore needs] -> Archive the full `/app/data` tree, not only Mealie-generated backup ZIPs, and validate restore contents during the recovery drill.
- [Mealie UI restore on PostgreSQL requires elevated privileges] -> Treat UI restore as a separate maintenance procedure with temporary role elevation, not the primary DR path.
- [Namespace conventions remain inconsistent across the repo] -> Scope this change to Mealie only and capture any broader standards cleanup as future work.
- [Shared backup account increases blast radius for storage-account-level issues] -> Keep workload data inside the dedicated `mealie` container, split database and filesystem artifacts by prefix, and scope Azure access through workload identity.

## Migration Plan

1. Add the Mealie application base and production overlay resources.
2. Add the Mealie CNPG cluster resources, workload identity-backed Azure backup integration, and scheduled backup.
3. Add Mealie file-backup resources and wire them into the production overlay.
4. Add Mealie and its CNPG cluster to the production Kustomizations.
5. Provision the shared Azure Blob backup container layout, the `id-mealie-backup` and `fic-mealie-backup` workload identity prerequisites, the `Storage Blob Data Contributor` role assignment on the `mealie` container, the OpenAI Key Vault secret, and DNS records needed by the manifests.
6. Reconcile through Flux after the change is committed and pushed.
7. Validate healthy deployment, ingress reachability, Homepage discovery, OpenAI configuration, CNPG backup execution, and file backup execution.
8. Run a recovery drill for CNPG and a separate restore drill for `/app/data`.

Rollback strategy:
- Remove Mealie application resources from the production Kustomization if deployment must be withdrawn.
- Remove Mealie database resources only after confirming whether any persisted state should be preserved.
- Keep Azure backup artifacts until rollback is complete and data retention decisions are made.

## Open Questions

- What Mealie hostname should be used for production ingress?
- What retention period should be used for CNPG backups and `/app/data` archives?
- Should the initial Mealie deployment include SMTP from day one, or leave it for a follow-up change?
- Should backup failure alerting be included now or captured as follow-up work once the first backup jobs are running?
