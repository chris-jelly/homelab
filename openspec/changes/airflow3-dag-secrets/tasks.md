## 1. Salesforce Org Setup (manual -- requires SF CLI and Setup UI)

- [x] 1.1 Generate RSA key pair (2048-bit, X.509 cert, 365-day validity)
- [x] 1.2 Upload `server.crt` to "integrations" Connected App via Setup > App Manager > Edit (not the External Client App UI)
- [x] 1.3 Verify OAuth scopes on "integrations" app include `api` and `refresh_token, offline_access`
- [x] 1.4 Create integration user: `Salesforce` license (full), `Read Only` profile, username `airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com`. Note: `Salesforce Integration` license was rejected -- does not permit access to standard CRM objects.
- [x] 1.5 Populate `Integration_App_Access` permission set: ViewAll on Account, Contact, Opportunity, Lead, Case, Campaign; Read on Product2, Pricebook2. OpportunityHistory/PricebookEntry/User not directly permissionable.
- [x] 1.6 Assign `Integration_App_Access` permission set to the integration user
- [x] 1.7 Store RSA private key in Azure KV (`kv-work-integrations`) as `app--salesforce-private-key--prod`
- [x] 1.8 Verify JWT Bearer flow: `sf org login jwt` succeeded -- authorized `airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com` with org ID `00DgK000001kz6OUAQ`

## 2. Warehouse PostgreSQL Connection ExternalSecret (agent-implementable)

- [x] 2.1 Create `apps/production/airflow/warehouse-postgres-conn-externalsecret.yaml` with ESO template composing `AIRFLOW_CONN_WAREHOUSE_POSTGRES` URI from `azure-kv-store` ClusterSecretStore and `app--warehouse-postgres-password--prod`

## 3. Salesforce Connection ExternalSecret (agent-implementable)

- [x] 3.1 Create `apps/production/airflow/salesforce-conn-externalsecret.yaml` with ESO template composing `AIRFLOW_CONN_SALESFORCE` JSON blob using JWT Bearer fields (`consumer_key`, `private_key`, `login`) from `azure-kv-work-integrations-store` ClusterSecretStore

## 4. Kustomization Update (agent-implementable)

- [x] 4.1 Add `warehouse-postgres-conn-externalsecret.yaml` and `salesforce-conn-externalsecret.yaml` to `apps/production/airflow/kustomization.yaml` resources list
- [x] 4.2 Remove `salesforce-credentials-externalsecret.yaml` and `postgres-credentials-externalsecret.yaml` from `apps/production/airflow/kustomization.yaml` resources list

## 5. Remove Legacy ExternalSecrets (agent-implementable)

- [x] 5.1 Delete `apps/production/airflow/salesforce-credentials-externalsecret.yaml`
- [x] 5.2 Delete `apps/production/airflow/postgres-credentials-externalsecret.yaml`

## 6. Verification

- [ ] 6.1 Confirm Flux reconciles new ExternalSecrets and K8s Secrets `warehouse-postgres-conn` and `salesforce-conn` exist in `airflow` namespace
- [ ] 6.2 Confirm `postgres_ping` DAG succeeds using `warehouse_postgres` conn_id
- [ ] 6.3 Confirm `salesforce_extraction` DAG succeeds using `salesforce` conn_id
