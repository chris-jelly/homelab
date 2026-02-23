## Why

Airflow is currently deployed on Helm chart 1.15.0 (Airflow 2.9.3). Chart 1.19.0 ships Airflow 3.1.7 — a major version jump that brings a new API server architecture, improved executor model, and numerous bug fixes accumulated over 4 chart releases. The existing Airflow state (DAG history, task logs, connections) is not valuable enough to warrant a careful in-place migration, so we will treat this as a clean redeploy: tear down the current release and database, then deploy fresh on 1.19.0.

Additionally, the CloudNative-PG operator is on Helm chart `0.24.0` and should be bumped to `0.27.1`. Since we're already tearing down and recreating the Airflow CNPG cluster, this is the ideal time to upgrade the operator with minimal risk.

## What Changes

- **BREAKING** Helm chart version bump from `1.15.0` → `1.19.0`
- **BREAKING** Airflow major version change from `2.9.3` → `3.1.7`
- **BREAKING** Airflow 3 replaces the webserver with a separate API server component — `webserver` Helm values become `apiServer` configuration
- **BREAKING** `webserver.defaultUser` settings move under the `createUserJob` section
- **BREAKING** Auth backend `airflow.api.auth.backend.basic_auth` removed in Airflow 3 — must update to new auth model
- **BREAKING** Probe paths changed for Airflow 3.x (health/liveness/readiness endpoints)
- **BREAKING** `[webserver] secret_key` replaced by `[api] secret_key` — chart now generates a JWT secret
- Clean teardown of the existing Airflow HelmRelease, CNPG database cluster, and metadata before redeployment
- Git-sync image updated from chart default to `4.4.2`
- Helm values restructured: Celery worker options moved under `workers.celery.*` (not relevant to us since we use KubernetesExecutor, but worth noting for values compatibility)
- Update `config` env vars to remove deprecated Airflow 2-specific settings (`AIRFLOW__API__AUTH_BACKENDS`, `AIRFLOW__KUBERNETES__*` namespace configs renamed in AF3)
- Re-validate resource limits, ingress, external secrets, and CNPG database connection after redeploy
- CloudNative-PG operator Helm chart bump from `0.24.0` → `0.27.1`

## Capabilities

### New Capabilities

- `airflow-deployment`: Covers the full Airflow Helm deployment configuration — chart version, image version, executor, git-sync, ingress, secrets, database connectivity, and Flux reconciliation settings. This spec does not exist today and should be created to document the target state.
- `cnpg-operator`: Covers the CloudNative-PG operator deployment — chart version, CRD management, and namespace configuration. This spec does not exist today and should be created to document the target state.

### Modified Capabilities

_(none — no existing specs have requirement-level changes)_

## Impact

- **apps/base/airflow/release.yaml**: Complete rewrite of HelmRelease values for chart 1.19.0 / Airflow 3 structure
- **apps/base/airflow/externalsecret.yaml**: Admin credentials env var names may change for Airflow 3 `createUserJob` format
- **apps/production/airflow/database-externalsecret.yaml**: Metadata connection secret format may need adjustment for AF3 db connection string requirements
- **databases/data/airflow/database.yaml**: CNPG cluster will be torn down and recreated (fresh database for AF3 schema)
- **apps/base/airflow/repository.yaml**: No change expected (same Helm repo)
- **apps/production/airflow/certificate.yaml**: No change expected
- **apps/production/airflow/salesforce-credentials-externalsecret.yaml**: No change expected
- **apps/production/airflow/postgres-credentials-externalsecret.yaml**: No change expected
- **DAGs repository** (`chris-jelly/de-airflow-pipeline`): DAGs may need updates for Airflow 3 API compatibility (out of scope for this change, but flagged as a dependency)
- **infrastructure/controllers/base/cloudnative-pg/release.yaml**: Bump chart version from `0.24.0` to `0.27.1`
- **Downtime**: Full Airflow outage during teardown and redeploy — acceptable for a homelab. Existing warehouse CNPG clusters managed by the operator should be unaffected by the operator upgrade.
