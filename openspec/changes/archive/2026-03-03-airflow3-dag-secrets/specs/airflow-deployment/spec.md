## REMOVED Requirements

### Requirement: Legacy Salesforce credentials ExternalSecret
**Reason**: The `salesforce-credentials` ExternalSecret produced individual `consumer_key` and `consumer_secret` keys. No Airflow 3 DAG references these keys — DAGs now use `conn_id`-based hooks with a single `AIRFLOW_CONN_SALESFORCE` JSON blob provided by the new `salesforce-conn` ExternalSecret. The Salesforce auth flow has also changed from consumer key/secret to JWT Bearer, making `consumer_secret` unnecessary.
**Migration**: The `salesforce-conn` ExternalSecret (in the `airflow-dag-connections` capability) replaces this secret. DAGs in `de-airflow-pipeline` already reference `conn_id="salesforce"` which maps to `AIRFLOW_CONN_SALESFORCE`.

### Requirement: Legacy PostgreSQL credentials ExternalSecret
**Reason**: The `postgres-credentials` ExternalSecret produced individual `host`, `database`, `username`, and `password` keys. No Airflow 3 DAG references these keys — DAGs now use `conn_id`-based hooks with a single `AIRFLOW_CONN_WAREHOUSE_POSTGRES` URI provided by the new `warehouse-postgres-conn` ExternalSecret.
**Migration**: The `warehouse-postgres-conn` ExternalSecret (in the `airflow-dag-connections` capability) replaces this secret. DAGs in `de-airflow-pipeline` already reference `conn_id="warehouse_postgres"` which maps to `AIRFLOW_CONN_WAREHOUSE_POSTGRES`.

## ADDED Requirements

### Requirement: Kustomization includes connection ExternalSecrets
The production Kustomization at `apps/production/airflow/kustomization.yaml` SHALL include resource entries for `warehouse-postgres-conn-externalsecret.yaml` and `salesforce-conn-externalsecret.yaml`.

#### Scenario: New ExternalSecret manifests are in the resource list
- **WHEN** the Kustomization manifest is applied
- **THEN** the `resources` list includes entries for the warehouse PostgreSQL connection and Salesforce connection ExternalSecret manifests

#### Scenario: Old ExternalSecret manifests are removed from the resource list
- **WHEN** the Kustomization manifest is applied
- **THEN** the `resources` list does NOT include `salesforce-credentials-externalsecret.yaml` or `postgres-credentials-externalsecret.yaml`
