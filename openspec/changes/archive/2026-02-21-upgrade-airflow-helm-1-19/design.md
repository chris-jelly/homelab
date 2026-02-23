## Context

Airflow runs on Helm chart 1.15.0 (Airflow 2.9.3) using KubernetesExecutor, an external CNPG PostgreSQL 15 database, git-sync for DAGs from `chris-jelly/de-airflow-pipeline`, Traefik ingress at `airflow.jellylabs.xyz`, and admin credentials injected via ExternalSecrets from Azure Key Vault. The metadata database connection is wired through a local Kubernetes SecretStore that reads the CNPG-generated `uri` field.

Four chart releases separate us from 1.19.0, with the most significant boundary being chart 1.17.0 which switched the default image from Airflow 2.x to 3.x. Since we don't care about preserving existing state, we can skip any migration complexity and do a clean cut-over.

The CloudNative-PG operator is on Helm chart `0.24.0` and manages three database clusters: `airflow-db-production-cnpg-v1` (PG 15), `warehouse-db-production-cnpg-v1` (PG 13), and `warehouse-db-dev-cnpg-v1` (PG 13). Since the Airflow CNPG cluster is being torn down anyway, this is a good time to bump the operator to `0.27.1`.

## Goals / Non-Goals

**Goals:**

- Upgrade to Helm chart 1.19.0 running Airflow 3.1.7
- Rewrite the HelmRelease values to match the 1.19.0 chart structure and Airflow 3 conventions
- Maintain the same deployment architecture: KubernetesExecutor, external CNPG database, git-sync DAGs, Traefik ingress, ExternalSecrets for credentials
- Create an `airflow-deployment` spec documenting the target state
- Minimise complexity by doing a clean teardown/redeploy rather than in-place migration
- Bump CloudNative-PG operator from `0.24.0` to `0.27.1` while the Airflow database is being recreated

**Non-Goals:**

- Preserving DAG run history, task logs, connections, or variables from the Airflow 2 instance
- Updating DAGs in `de-airflow-pipeline` for Airflow 3 compatibility (separate effort)
- Upgrading PostgreSQL version (staying on PG 15)
- Enabling new chart features beyond what we currently use (no Celery workers, PgBouncer, StatsD, KEDA, etc.)
- Zero-downtime deployment
- Changing the warehouse or warehouse-dev CNPG clusters (they should be unaffected by the operator bump)

## Decisions

### 1. Clean teardown and redeploy vs. in-place upgrade

**Choice**: Clean teardown and redeploy.

**Rationale**: An in-place upgrade from Airflow 2.9 → 3.1 requires running `airflow db migrate`, handling schema changes, and dealing with potential DAG parsing failures during the transition. Since the homelab Airflow state has no production value, tearing everything down and redeploying avoids all migration edge cases.

**Alternatives considered**:
- *In-place Helm upgrade*: Would require careful Airflow 2→3 db migration, DAG compatibility testing before cutover, and dealing with potential chart values that changed shape between 4 releases. Too much complexity for no benefit here.

### 2. Teardown sequence: Git-driven via Flux

**Choice**: Remove the Airflow app manifests from the Flux kustomization (or suspend the HelmRelease), let Flux reconcile to tear down the release, then drop and recreate the CNPG database cluster, then re-add the updated manifests.

**Approach**:
1. Suspend the Airflow HelmRelease (`flux suspend hr airflow -n airflow`) to prevent reconciliation during changes
2. Delete the HelmRelease resource manually (`flux delete hr airflow -n airflow`) so Helm uninstalls the release
3. Delete the CNPG Cluster resource to drop the database (`kubectl delete cluster airflow-db-production-cnpg-v1 -n airflow`)
4. Commit the updated manifests (new chart version, new values)
5. Let Flux reconcile — it will recreate the HelmRelease and CNPG cluster with fresh state

**Rationale**: This keeps the GitOps flow intact. The CNPG cluster must be explicitly deleted because simply changing the HelmRelease won't touch it — it's managed by a separate kustomization. Recreating it gives us a clean Airflow 3 schema from the `migrateDatabaseJob`.

**Alternatives considered**:
- *Just bump the chart version in-place*: Risky — the Airflow 2 database schema would confuse Airflow 3's migration job, and old Helm state might conflict with renamed resources.
- *Delete namespace entirely*: Overkill — would destroy the TLS certificate, ExternalSecrets, and RBAC resources that don't need to change.

### 3. Airflow 3 API server replaces webserver

**Choice**: Replace `webserver` values section with `apiServer` configuration. Move `webserver.defaultUser` to `createUserJob`.

**Rationale**: Airflow 3 splits the webserver into a standalone API server. The Helm chart 1.17+ exposes this as `apiServer` in values. The `webserver.defaultUser` was deprecated in 1.19.0 and moved to `createUserJob`. Since the chart still generates the JWT secret automatically, we don't need to manage it ourselves.

**Key values changes**:
```yaml
# REMOVE
webserver:
  defaultUser:
    enabled: true
  resources: ...

# ADD
apiServer:
  resources:
    requests:
      memory: 1Gi
      cpu: 200m
    limits:
      memory: 2Gi
      cpu: 1000m

createUserJob:
  useHelmHooks: false
  applyCustomEnv: true
  defaultUser:
    enabled: true
```

### 4. Ingress changes for Airflow 3

**Choice**: Move from `ingress.web` to `ingress.apiServer` for the UI/API endpoint.

**Rationale**: In Airflow 3, the UI is served by the API server, not the webserver. The Helm chart 1.17+ reflects this in the ingress configuration keys.

```yaml
# REMOVE
ingress:
  web:
    enabled: true
    ...

# ADD
ingress:
  apiServer:
    enabled: true
    ingressClassName: traefik
    hosts:
      - name: airflow.jellylabs.xyz
        tls:
          enabled: true
          secretName: airflow-cert-secret
```

### 5. Config env var cleanup

**Choice**: Remove deprecated Airflow 2 config variables and update to Airflow 3 equivalents.

**Changes**:
- **Remove** `AIRFLOW__API__AUTH_BACKENDS`: Airflow 3 uses a new auth system (FAB-based auth is still default for the UI, but the API backend config key is gone)
- **Remove** `AIRFLOW__WEBSERVER__EXPOSE_CONFIG`: Replaced by API server config exposure in AF3
- **Keep** `AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION` and `AIRFLOW__CORE__LOAD_EXAMPLES` (still valid in AF3)
- **Keep** `AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK` (still valid)
- **Update** KubernetesExecutor config: `AIRFLOW__KUBERNETES__*` keys are still supported in AF3 via the `kubernetes` provider config section — keep as-is

### 6. Admin credentials ExternalSecret

**Choice**: Keep the same ExternalSecret structure. The `_AIRFLOW_WWW_USER_*` environment variables are still consumed by the `createUserJob` in chart 1.19.0.

**Rationale**: The chart's `createUserJob` still reads these env vars to create the initial admin user. The ExternalSecret that injects them via `extraEnvFrom` remains valid. No changes needed to `externalsecret.yaml` or Azure Key Vault secrets.

### 7. Database connection secret format

**Choice**: Keep the existing `airflow-metadata-connection` ExternalSecret and CNPG secret-reading mechanism unchanged.

**Rationale**: The chart's `data.metadataSecretName` still expects a secret with a `connection` key containing the PostgreSQL URI. CNPG still generates the `uri` property in its app secret. The plumbing doesn't change — only the CNPG Cluster resource gets deleted and recreated to give Airflow 3 a fresh schema.

### 8. Git-sync configuration

**Choice**: Keep the same git-sync config, but remove `containerName` and `uid` overrides since the chart 1.19.0 defaults (git-sync 4.4.2) handle these correctly.

**Rationale**: The git-sync image bump from the chart-default (which was ~3.6.x in 1.15.0) to 4.4.2 is handled automatically by the chart. The `containerName: git-sync` and `uid: 65533` were workarounds for older git-sync versions. The new git-sync 4.x uses different conventions. Keeping `repo`, `branch`, `subPath`, `wait`, and resource limits is sufficient.

### 9. CloudNative-PG operator upgrade

**Choice**: Bump the CNPG operator Helm chart from `0.24.0` to `0.27.1` as part of this change.

**Rationale**: The operator manages all CNPG database clusters in the cluster. Normally, upgrading the operator mid-flight carries some risk to running databases. However, since we're already tearing down and recreating the Airflow CNPG cluster, the window of disruption overlaps. The warehouse clusters (PG 13, single-instance) should be unaffected — CNPG operator upgrades are backward-compatible with existing Cluster resources, and the operator simply reconciles them with the new version.

**Sequencing**: Bump the operator version *before* the Airflow teardown. This ensures the new operator is running when the fresh Airflow CNPG cluster is created, so it gets managed by the new operator from the start.

**Change**: Single line in `infrastructure/controllers/base/cloudnative-pg/release.yaml`:
```yaml
version: 0.24.0  →  version: 0.27.1
```

**Alternatives considered**:
- *Separate change*: Could upgrade the operator independently, but since we're already disrupting the Airflow database, combining them reduces the number of rolling changes and the operator upgrade is a single version bump.

## Risks / Trade-offs

- **[DAG compatibility]** → Airflow 3 has breaking changes in DAG authoring APIs (e.g., `TaskFlow` changes, removed operators). DAGs in `de-airflow-pipeline` may fail to parse on AF3. **Mitigation**: Flag as a separate prerequisite task; the Airflow deployment itself will work but DAGs may need updating.

- **[Downtime during cutover]** → Airflow will be completely unavailable during teardown and redeploy. **Mitigation**: Acceptable for homelab; schedule during low-usage period. No external dependencies are affected (DAGs are batch workloads).

- **[CNPG data loss]** → Deleting the CNPG Cluster resource destroys the database and its PVC. **Mitigation**: This is intentional — we explicitly don't want the old data. But if someone later wants historical data, it's gone. Document this decision.

- **[ExternalSecret race condition]** → If Flux reconciles the HelmRelease before ExternalSecrets are ready, the deployment may fail on first attempt. **Mitigation**: The existing `apps.yaml` Flux Kustomization already depends on `databases`, and ExternalSecrets are part of the same kustomization as the HelmRelease. Flux remediation retries (3 attempts) handle transient ordering issues.

- **[Chart values drift]** → Jumping 4 chart versions means undocumented values changes may exist beyond the release notes. **Mitigation**: Use `helm template` to validate rendered manifests before committing. The clean redeploy approach means any issues surface immediately rather than corrupting existing state.

- **[CNPG operator upgrade affecting warehouse clusters]** → The operator bump from 0.24.0 → 0.27.1 could theoretically cause issues with the warehouse CNPG clusters (PG 13). **Mitigation**: CNPG operator upgrades are designed to be backward-compatible. The operator simply reconciles existing Cluster resources. Single-instance clusters with no backups configured are the simplest case. Monitor the warehouse pods after the operator upgrade lands.

## Resolved Questions

- **Image tag pinning**: No — rely on the chart default. This is a homelab; accepting the chart's bundled Airflow version on each chart upgrade is fine and reduces maintenance overhead.
- **DAG compatibility sequencing**: Deploy Airflow 3 first, fix DAGs after. The deployment will be healthy even if DAGs fail to parse — the scheduler will just log import errors. DAG updates are a separate follow-up task.
