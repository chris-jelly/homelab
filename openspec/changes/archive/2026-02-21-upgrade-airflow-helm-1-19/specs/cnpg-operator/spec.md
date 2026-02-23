## ADDED Requirements

### Requirement: CNPG operator Helm chart version
The CloudNative-PG operator HelmRelease SHALL use Helm chart version `0.27.1` from the `https://cloudnative-pg.github.io/charts` HelmRepository.

#### Scenario: Chart version is correct
- **WHEN** the CNPG HelmRelease is reconciled by Flux
- **THEN** the chart spec version is `0.27.1` and sourceRef points to the `cnpg` HelmRepository in `flux-system`

### Requirement: CRD management
The operator SHALL install and manage CRDs via the Helm chart. CRDs SHALL be preserved on chart uninstall.

#### Scenario: CRDs are enabled and kept
- **WHEN** the HelmRelease values are applied
- **THEN** `crds.enabled` is `true` and `crds.keep` is `true`

### Requirement: Operator namespace
The CNPG operator SHALL be deployed in the `cloudnative-pg` namespace.

#### Scenario: Namespace and target are correct
- **WHEN** the HelmRelease manifest is applied
- **THEN** `metadata.namespace` is `cloudnative-pg` and `spec.targetNamespace` is `cloudnative-pg`

### Requirement: Backward compatibility with existing clusters
The operator upgrade SHALL NOT disrupt existing CNPG Cluster resources. The warehouse and warehouse-dev database clusters SHALL continue running without intervention after the operator upgrade.

#### Scenario: Warehouse clusters remain healthy after upgrade
- **WHEN** the CNPG operator is upgraded from `0.24.0` to `0.27.1`
- **THEN** the `warehouse-db-production-cnpg-v1` and `warehouse-db-dev-cnpg-v1` Cluster resources remain in a healthy state and their pods are not restarted unnecessarily

### Requirement: Operator upgrade before Airflow teardown
The CNPG operator upgrade SHALL be applied before the Airflow CNPG cluster is torn down and recreated, so the new Airflow database cluster is managed by the updated operator from creation.

#### Scenario: Sequencing of operator upgrade and Airflow teardown
- **WHEN** the upgrade process is executed
- **THEN** the CNPG operator HelmRelease version bump is committed and reconciled before the Airflow CNPG Cluster resource is deleted and recreated
