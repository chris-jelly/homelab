## ADDED Requirements

### Requirement: Warehouse PostgreSQL connection secret
The system SHALL provision a Kubernetes Secret named `warehouse-postgres-conn` in the `airflow` namespace via an ExternalSecret. The secret SHALL contain a single key `AIRFLOW_CONN_WAREHOUSE_POSTGRES` with a PostgreSQL connection URI composed from an ESO template. The URI SHALL use the format `postgresql://app:<password>@warehouse-db-production-cnpg-v1-rw.warehouse.svc.cluster.local:5432/app` where `<password>` is sourced from Azure Key Vault secret `app--warehouse-postgres-password--prod` via the `azure-kv-store` ClusterSecretStore.

#### Scenario: ExternalSecret creates the warehouse connection secret
- **WHEN** the ExternalSecret `warehouse-postgres-conn` reconciles
- **THEN** a Kubernetes Secret named `warehouse-postgres-conn` exists in the `airflow` namespace with a single key `AIRFLOW_CONN_WAREHOUSE_POSTGRES` containing a valid PostgreSQL URI

#### Scenario: URI is composed from ESO template
- **WHEN** the ExternalSecret template is rendered
- **THEN** the URI embeds the password from Azure KV secret `app--warehouse-postgres-password--prod` and hardcodes host `warehouse-db-production-cnpg-v1-rw.warehouse.svc.cluster.local`, port `5432`, database `app`, and username `app`

#### Scenario: Secret store reference is correct
- **WHEN** the ExternalSecret manifest is applied
- **THEN** `secretStoreRef` references `azure-kv-store` ClusterSecretStore

### Requirement: Salesforce connection secret
The system SHALL provision a Kubernetes Secret named `salesforce-conn` in the `airflow` namespace via an ExternalSecret. The secret SHALL contain a single key `AIRFLOW_CONN_SALESFORCE` with a JSON blob for the Airflow Salesforce provider's JWT Bearer OAuth flow. The ExternalSecret SHALL use the `azure-kv-work-integrations-store` ClusterSecretStore.

#### Scenario: ExternalSecret creates the Salesforce connection secret
- **WHEN** the ExternalSecret `salesforce-conn` reconciles
- **THEN** a Kubernetes Secret named `salesforce-conn` exists in the `airflow` namespace with a single key `AIRFLOW_CONN_SALESFORCE` containing a valid JSON blob

#### Scenario: JSON blob uses JWT Bearer auth fields
- **WHEN** the ExternalSecret template is rendered
- **THEN** the JSON blob SHALL contain `conn_type` set to `salesforce`, `login` set to the integration user's username, and an `extra` object containing `consumer_key` (from Azure KV secret `app--salesforce-consumer-key--prod`), `private_key` (from Azure KV secret `app--salesforce-private-key--prod`), and `domain` set to `login`

#### Scenario: consumer_secret is not referenced
- **WHEN** the ExternalSecret manifest is applied
- **THEN** the ExternalSecret SHALL NOT reference the Azure KV secret `app--salesforce-consumer-secret--prod` because JWT Bearer flow does not use it

#### Scenario: Secret store reference is correct
- **WHEN** the ExternalSecret manifest is applied
- **THEN** `secretStoreRef` references `azure-kv-work-integrations-store` ClusterSecretStore

### Requirement: ExternalSecret refresh interval
All connection ExternalSecrets SHALL use a `refreshInterval` of `1h` to sync changes from Azure Key Vault.

#### Scenario: Refresh interval is set
- **WHEN** any connection ExternalSecret manifest is applied
- **THEN** `spec.refreshInterval` is `1h`

### Requirement: ExternalSecret ownership policy
All connection ExternalSecrets SHALL use `creationPolicy: Owner` so that the generated Kubernetes Secret is garbage-collected when the ExternalSecret is deleted.

#### Scenario: Ownership policy is set
- **WHEN** any connection ExternalSecret manifest is applied
- **THEN** `spec.target.creationPolicy` is `Owner`

### Requirement: Salesforce org prerequisites
Before the `salesforce-conn` ExternalSecret can produce a functional connection, the following Salesforce org configuration SHALL be completed manually (outside of GitOps):

1. An RSA key pair SHALL be generated (2048-bit, X.509 certificate, 365-day validity)
2. The public certificate SHALL be uploaded to the "integrations" Connected App via Setup > App Manager (the External Client App UI does not expose the certificate upload field)
3. A dedicated integration user SHALL be created using the `Salesforce` license (full) and `Read Only` profile, with username `airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com`. Note: the `Salesforce Integration` license was evaluated but does not permit access to standard CRM objects.
4. The `Integration_App_Access` permission set SHALL be populated with ViewAll on: Account, Contact, Opportunity, Lead, Case, Campaign; and Read on: Product2, Pricebook2. OpportunityHistory, PricebookEntry, and User are not directly permissionable.
5. The permission set SHALL be assigned to the integration user (which also pre-authorizes them for the Connected App)
6. The RSA private key SHALL be stored in Azure Key Vault (`kv-work-integrations`) as secret `app--salesforce-private-key--prod`

#### Scenario: JWT Bearer flow is testable after prerequisites
- **WHEN** all six prerequisites are completed
- **THEN** `sf org login jwt --client-id <consumer_key> --jwt-key-file <key> --username airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com` SHALL succeed against the Salesforce org
