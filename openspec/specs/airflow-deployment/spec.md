## ADDED Requirements

### Requirement: Helm chart version and Airflow image
The Airflow HelmRelease SHALL use Helm chart version `1.19.0` from the `https://airflow.apache.org` HelmRepository. The chart's default Airflow image SHALL be used (no `defaultAirflowTag` override).

#### Scenario: Chart version is correct
- **WHEN** the HelmRelease is reconciled by Flux
- **THEN** the chart spec version is `1.19.0` and sourceRef points to the `airflow` HelmRepository in `flux-system`

#### Scenario: Default image is used
- **WHEN** the HelmRelease values are rendered
- **THEN** no `defaultAirflowRepository` or `defaultAirflowTag` overrides are present, and the chart's bundled Airflow 3.x image is used

### Requirement: KubernetesExecutor
The deployment SHALL use `KubernetesExecutor` as the sole executor. Celery, Redis, Flower, and PgBouncer components SHALL be disabled.

#### Scenario: Executor is KubernetesExecutor
- **WHEN** the HelmRelease values are applied
- **THEN** `executor` is set to `KubernetesExecutor`

#### Scenario: Unused components are disabled
- **WHEN** the HelmRelease values are applied
- **THEN** `flower.enabled`, `redis.enabled`, and `pgbouncer.enabled` are all `false`

### Requirement: External CNPG database
The deployment SHALL use an external CloudNative-PG PostgreSQL 15 cluster as the metadata database. The built-in PostgreSQL subchart SHALL be disabled. The metadata connection SHALL be provided via a Kubernetes secret named `airflow-metadata-connection` with a `connection` key containing the PostgreSQL URI.

#### Scenario: Built-in PostgreSQL is disabled
- **WHEN** the HelmRelease values are applied
- **THEN** `postgresql.enabled` is `false`

#### Scenario: Metadata secret is referenced
- **WHEN** the HelmRelease values are applied
- **THEN** `data.metadataSecretName` is set to `airflow-metadata-connection`

#### Scenario: CNPG cluster provides the connection URI
- **WHEN** the ExternalSecret `airflow-metadata-connection` reconciles
- **THEN** it reads the `uri` property from the CNPG-generated secret `airflow-db-production-cnpg-v1-app` and maps it to the `connection` key

### Requirement: Git-sync DAG delivery
The deployment SHALL use git-sync to pull DAGs from `https://github.com/chris-jelly/de-airflow-pipeline`, branch `main`, subpath `dags`. The git-sync sidecar SHALL have resource limits configured.

#### Scenario: Git-sync is enabled with correct repo
- **WHEN** the HelmRelease values are applied
- **THEN** `dags.gitSync.enabled` is `true`, `repo` is `https://github.com/chris-jelly/de-airflow-pipeline`, `branch` is `main`, and `subPath` is `dags`

#### Scenario: Git-sync resource limits are set
- **WHEN** the HelmRelease values are applied
- **THEN** git-sync container has memory request `64Mi`, memory limit `128Mi`, cpu request `50m`, cpu limit `100m`

#### Scenario: Legacy git-sync overrides are removed
- **WHEN** the HelmRelease values are applied
- **THEN** `containerName` and `uid` overrides are NOT present in `dags.gitSync` (chart 1.19.0 defaults handle these)

### Requirement: API server replaces webserver
The deployment SHALL configure `apiServer` instead of `webserver` for the Airflow 3 UI/API component. No `webserver` section SHALL exist in the HelmRelease values.

#### Scenario: API server resources are configured
- **WHEN** the HelmRelease values are applied
- **THEN** `apiServer.resources` has memory request `1Gi`, memory limit `2Gi`, cpu request `200m`, cpu limit `1000m`

#### Scenario: No webserver section exists
- **WHEN** the HelmRelease values are rendered
- **THEN** there is no `webserver` key in the values

### Requirement: Ingress via Traefik
The deployment SHALL expose the Airflow UI via a Traefik ingress at `airflow.jellylabs.xyz` with TLS using the `airflow-cert-secret` certificate. The ingress SHALL use `ingress.apiServer` (not `ingress.web`).

#### Scenario: API server ingress is configured
- **WHEN** the HelmRelease values are applied
- **THEN** `ingress.apiServer.enabled` is `true`, `ingressClassName` is `traefik`, host name is `airflow.jellylabs.xyz`, and TLS uses secret `airflow-cert-secret`

#### Scenario: Legacy web ingress is not present
- **WHEN** the HelmRelease values are rendered
- **THEN** there is no `ingress.web` key in the values

### Requirement: Admin user creation
The deployment SHALL create a default admin user via the `createUserJob`. Admin credentials SHALL be injected from the `airflow-admin-credentials` ExternalSecret using `_AIRFLOW_WWW_USER_*` environment variables.

#### Scenario: createUserJob is configured for Flux
- **WHEN** the HelmRelease values are applied
- **THEN** `createUserJob.useHelmHooks` is `false`, `createUserJob.applyCustomEnv` is `true`, and `createUserJob.defaultUser.enabled` is `true`

#### Scenario: Admin credentials are injected
- **WHEN** the HelmRelease values are applied
- **THEN** `extraEnvFrom` includes a `secretRef` to `airflow-admin-credentials`

### Requirement: Database migration job for Flux
The `migrateDatabaseJob` SHALL have Helm hooks disabled for Flux compatibility.

#### Scenario: migrateDatabaseJob is configured for Flux
- **WHEN** the HelmRelease values are applied
- **THEN** `migrateDatabaseJob.useHelmHooks` is `false` and `migrateDatabaseJob.applyCustomEnv` is `true`

### Requirement: Airflow configuration variables
The deployment SHALL set core Airflow config via environment variables. Deprecated Airflow 2-specific config keys SHALL be removed.

#### Scenario: Core config is set
- **WHEN** the HelmRelease values are applied
- **THEN** `AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION` is `"true"` and `AIRFLOW__CORE__LOAD_EXAMPLES` is `"false"`

#### Scenario: Scheduler health check is enabled
- **WHEN** the HelmRelease values are applied
- **THEN** `AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK` is `"true"`

#### Scenario: KubernetesExecutor config is set
- **WHEN** the HelmRelease values are applied
- **THEN** `AIRFLOW__KUBERNETES__NAMESPACE` is `"airflow"`, `DELETE_WORKER_PODS` is `"True"`, `DELETE_WORKER_PODS_ON_FAILURE` is `"False"`, and `WORKER_PODS_CREATION_BATCH_SIZE` is `"5"`

#### Scenario: Deprecated Airflow 2 config is removed
- **WHEN** the HelmRelease values are rendered
- **THEN** `AIRFLOW__API__AUTH_BACKENDS` and `AIRFLOW__WEBSERVER__EXPOSE_CONFIG` are NOT present

### Requirement: Scheduler resource limits
The scheduler SHALL have resource requests and limits configured.

#### Scenario: Scheduler resources are set
- **WHEN** the HelmRelease values are applied
- **THEN** scheduler has memory request `512Mi`, memory limit `1Gi`, cpu request `200m`, cpu limit `1000m`

### Requirement: Worker pod defaults
KubernetesExecutor worker pods SHALL have resource limits configured. Worker log persistence SHALL be disabled (git-sync provides DAGs).

#### Scenario: Worker resources are set
- **WHEN** the HelmRelease values are applied
- **THEN** workers have memory request `512Mi`, memory limit `1Gi`, cpu request `200m`, cpu limit `1000m`

#### Scenario: Worker persistence is disabled
- **WHEN** the HelmRelease values are applied
- **THEN** `workers.persistence.enabled` is `false`

### Requirement: Flux reconciliation settings
The HelmRelease SHALL reconcile every 10 minutes with install and upgrade remediation retries set to 3.

#### Scenario: Reconciliation interval and retries
- **WHEN** the HelmRelease manifest is applied
- **THEN** `spec.interval` is `10m`, `spec.install.remediation.retries` is `3`, and `spec.upgrade.remediation.retries` is `3`

### Requirement: Clean teardown before deployment
The upgrade SHALL be performed as a clean teardown and redeploy. The existing HelmRelease and CNPG database cluster SHALL be deleted before the new manifests are applied. No Airflow 2 data migration SHALL be performed.

#### Scenario: Old HelmRelease is removed
- **WHEN** the upgrade process begins
- **THEN** the existing Airflow HelmRelease is deleted (via `flux delete hr` or suspension + removal)

#### Scenario: CNPG cluster is recreated
- **WHEN** the upgrade process begins
- **THEN** the existing `airflow-db-production-cnpg-v1` CNPG Cluster resource is deleted, and a new one is created when Flux reconciles the updated manifests

#### Scenario: Fresh database schema
- **WHEN** the new HelmRelease is reconciled
- **THEN** the `migrateDatabaseJob` runs against an empty database, creating the Airflow 3 schema from scratch
