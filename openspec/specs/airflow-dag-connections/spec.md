## Requirements

### Requirement: Warehouse PostgreSQL connection secret
The system SHALL provision a Kubernetes Secret named `warehouse-postgres-conn` in the `airflow` namespace via an ExternalSecret. The secret SHALL contain a single key `AIRFLOW_CONN_WAREHOUSE_POSTGRES` with the PostgreSQL connection URI sourced directly from the CNPG-generated secret `warehouse-db-production-cnpg-v1-app` in the `warehouse` namespace via the `kubernetes-warehouse` SecretStore. The ExternalSecret SHALL read the `uri` property from the CNPG secret without composition or templating.

#### Scenario: ExternalSecret creates the warehouse connection secret
- **WHEN** the ExternalSecret `warehouse-postgres-conn` reconciles
- **THEN** a Kubernetes Secret named `warehouse-postgres-conn` exists in the `airflow` namespace with a single key `AIRFLOW_CONN_WAREHOUSE_POSTGRES` containing a valid PostgreSQL URI

#### Scenario: URI is sourced directly from CNPG secret
- **WHEN** the ExternalSecret reads from the `kubernetes-warehouse` SecretStore
- **THEN** it maps `remoteRef.key` `warehouse-db-production-cnpg-v1-app` with `remoteRef.property` `uri` to `secretKey` `AIRFLOW_CONN_WAREHOUSE_POSTGRES`

#### Scenario: Secret store reference uses Kubernetes provider
- **WHEN** the ExternalSecret manifest is applied
- **THEN** `secretStoreRef` references `kubernetes-warehouse` SecretStore (not a ClusterSecretStore or Azure KV store)

#### Scenario: No ESO template composition is used
- **WHEN** the ExternalSecret manifest is applied
- **THEN** there is no `spec.target.template` section -- the URI is passed through directly from the CNPG secret

#### Scenario: Refresh interval matches in-cluster pattern
- **WHEN** the ExternalSecret manifest is applied
- **THEN** `spec.refreshInterval` is `30s` (matching the Airflow metadata connection ExternalSecret pattern for in-cluster sources)

#### Scenario: Azure KV is not referenced
- **WHEN** the ExternalSecret manifest is applied
- **THEN** the ExternalSecret does NOT reference `azure-kv-store` ClusterSecretStore or Azure KV secret `app--warehouse-postgres-password--prod`
