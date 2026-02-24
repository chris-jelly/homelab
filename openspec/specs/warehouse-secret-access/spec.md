## Requirements

### Requirement: Cross-namespace RBAC for warehouse secret access
The system SHALL define a Role in the `warehouse` namespace granting `get`, `list`, and `watch` on `secrets` resources and `create` on `selfsubjectrulesreviews`. A RoleBinding in the `warehouse` namespace SHALL bind the `external-secrets-kubernetes-reader` ServiceAccount from the `airflow` namespace to this Role.

#### Scenario: Role grants minimal secret read access
- **WHEN** the Role `external-secrets-warehouse-reader` is applied in the `warehouse` namespace
- **THEN** it grants `get`, `list`, `watch` on core API group `secrets` and `create` on `authorization.k8s.io` `selfsubjectrulesreviews`

#### Scenario: RoleBinding binds the airflow SA
- **WHEN** the RoleBinding `airflow-external-secrets-warehouse-reader` is applied in the `warehouse` namespace
- **THEN** it binds `ServiceAccount/external-secrets-kubernetes-reader` from `namespace: airflow` to the `external-secrets-warehouse-reader` Role

#### Scenario: No new ServiceAccount is created
- **WHEN** the RBAC resources are applied
- **THEN** the existing `external-secrets-kubernetes-reader` ServiceAccount in the `airflow` namespace is reused

### Requirement: Kubernetes SecretStore for warehouse namespace
The system SHALL define a SecretStore named `kubernetes-warehouse` in the `airflow` namespace using the ESO Kubernetes provider with `remoteNamespace: warehouse`. The SecretStore SHALL reference the `external-secrets-kubernetes-reader` ServiceAccount for authentication.

#### Scenario: SecretStore targets warehouse namespace
- **WHEN** the SecretStore `kubernetes-warehouse` is applied in the `airflow` namespace
- **THEN** `spec.provider.kubernetes.remoteNamespace` is `warehouse`

#### Scenario: SecretStore uses existing ServiceAccount
- **WHEN** the SecretStore `kubernetes-warehouse` is applied
- **THEN** `spec.provider.kubernetes.auth.serviceAccount.name` is `external-secrets-kubernetes-reader`

#### Scenario: SecretStore includes CA provider
- **WHEN** the SecretStore `kubernetes-warehouse` is applied
- **THEN** `spec.provider.kubernetes.server.caProvider` references ConfigMap `kube-root-ca.crt` key `ca.crt`

#### Scenario: SecretStore validates successfully
- **WHEN** the SecretStore is reconciled by ESO
- **THEN** the status condition shows `Ready: True` with reason `Valid`

### Requirement: RBAC resources owned by consumer
The RBAC resources (Role and RoleBinding in `warehouse` namespace) and the SecretStore SHALL be defined in the airflow application directory (`apps/production/airflow/`), not in the warehouse database directory.

#### Scenario: Resources are in the airflow app directory
- **WHEN** the manifests are checked into the repository
- **THEN** the Role, RoleBinding, and SecretStore are in a file under `apps/production/airflow/`

### Requirement: Pattern extensibility for future consumers
The Role in the `warehouse` namespace SHALL be named generically (`external-secrets-warehouse-reader`) so that future consumers can add their own RoleBindings to the same Role without modifying it.

#### Scenario: Future consumer adds access
- **WHEN** a new namespace (e.g., `streamlit`) needs warehouse secret access
- **THEN** only a new RoleBinding in the `warehouse` namespace and a new SecretStore in the consumer namespace are required -- the existing Role is reused
