# 004 - Warehouse Access Reconciliation

Runbook and contract for warehouse role management, privilege reconciliation, and admin login secret sourcing.

## Role Naming Contract

Production (`warehouse` namespace):
- Read role for Airflow: `warehouse_airflow_reader_prod`
- Read role for Streamlit: `warehouse_streamlit_reader_prod`
- Admin LOGIN role for pgAdmin: `warehouse_admin_prod`

The CNPG cluster manifest declaratively manages these roles. Password material for the admin role is sourced via External Secrets and attached through CNPG `passwordSecret`.

## Azure Key Vault Contract

- Cluster secret store: `azure-kv-store`
- Vault URL: `https://kv-jellyhomelabprod.vault.azure.net/`
- Expected Key Vault key for admin password: `secret/warehouse-admin-password-prod`

The ExternalSecret writes to Kubernetes secret `warehouse-admin-credentials` in `warehouse` namespace with:
- `username`: fixed to `warehouse_admin_prod`
- `password`: sourced from `secret/warehouse-admin-password-prod`

## Reconciler Contract

- Reconciler runs once at deploy (`Job`) and continuously (`CronJob`, every 15 minutes).
- Reconciler input is controlled by environment variables:
  - `DATABASE_URL`
  - `WRITER_ROLE`
  - `APPROVED_SCHEMAS`
  - `READER_ROLES`
- Current defaults:
  - `WRITER_ROLE=app`
  - `APPROVED_SCHEMAS=public`
  - `READER_ROLES=warehouse_airflow_reader_prod,warehouse_streamlit_reader_prod`

The reconciler grants:
- `CONNECT` on database for each reader role
- `USAGE` on each approved schema
- `SELECT` on all existing tables/views in each approved schema
- `USAGE, SELECT` on all existing sequences in each approved schema
- `ALTER DEFAULT PRIVILEGES` for the writer role so future tables/sequences inherit read access

## Validation Checklist

### 1) Bootstrap ordering and no-op safety

- Confirm CNPG managed roles reconcile first:
  - `kubectl -n warehouse get cluster warehouse-db-production-cnpg-v1 -o yaml`
- Run reconciler before objects exist and confirm no failure:
  - `kubectl -n warehouse logs job/warehouse-privilege-reconciler-bootstrap`
- Confirm logs show missing schema skip behavior when schema is absent.

### 2) Airflow read access validation

- Connect as Airflow read role and run:
  - `SELECT current_user;`
  - `SELECT count(*) FROM <approved_schema>.<existing_table>;`
- Create a new DAG-managed object and verify Airflow can read it without manual GRANT.

### 3) Streamlit read access validation

- Connect as Streamlit read role and run:
  - `SELECT current_user;`
  - `SELECT count(*) FROM <approved_schema>.<existing_table>;`
- Create a new DAG-managed object and verify Streamlit can read it without manual GRANT.

### 4) pgAdmin admin LOGIN validation

- Verify ExternalSecret synchronization:
  - `kubectl -n warehouse get externalsecret warehouse-admin-credentials`
- Login in pgAdmin with `warehouse_admin_prod` and confirm connection to `-rw` service endpoint.
- Validate admin actions stay within approved operational boundaries.

## Troubleshooting

- Reconciler failures:
  - `kubectl -n <ns> get jobs,cronjobs`
  - `kubectl -n <ns> logs job/<failed-job-name>`
- Secret sync failures:
  - `kubectl -n <ns> describe externalsecret warehouse-admin-credentials`
  - Confirm key exists in `kv-jellyhomelabprod` with exact key name.
- CNPG role drift:
  - `kubectl -n <ns> get cluster <cluster-name> -o yaml`
  - Verify `spec.managed.roles` includes expected principals.

## Drift Detection Checks

- Reader role cannot read expected schema/table -> run reconciler job manually and inspect logs.
- Newly created DAG object not readable -> verify `WRITER_ROLE` matches actual object creator role.
- Repeated CronJob failures -> check DB endpoint, DNS, and credentials from `<cluster>-app` secret.

## Rollback Notes

- Suspend periodic reconciliation:
  - `kubectl -n <ns> patch cronjob warehouse-privilege-reconciler -p '{"spec":{"suspend":true}}'`
- Revert manifests in Git and let Flux reconcile.
- If needed, remove managed roles from CNPG manifest in a follow-up PR and apply explicit temporary GRANT statements manually during incident response.
