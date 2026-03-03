## Why

Warehouse roles can be created declaratively, but table and schema privileges cannot be granted reliably at cluster bootstrap because DAG-managed objects do not exist yet. We need a durable, GitOps-friendly privilege reconciliation model that keeps read access current for Airflow and Streamlit as schemas and tables are created over time.

## What Changes

- Add a warehouse privilege reconciler pattern that continuously converges grants for consumer roles instead of relying on one-time bootstrap SQL.
- Define a read-only access contract for Streamlit and Airflow that includes database, schema, and object-level privileges for existing and future objects.
- Add declarative CNPG-managed roles for warehouse read-only access and a dedicated admin LOGIN role for human pgAdmin operations.
- Define credential sourcing and rotation for LOGIN roles using External Secrets + Azure Key Vault, while keeping privilege ownership in-database.
- Specify validation and drift-detection expectations so permission regressions are surfaced quickly after DAG schema evolution.

## Capabilities

### New Capabilities
- `warehouse-privilege-reconciliation`: Convergent grant/default-privilege model for DAG-created warehouse schemas and objects, including Streamlit and Airflow read paths.
- `warehouse-admin-login-access`: Admin LOGIN access pattern for pgAdmin with least-privilege defaults and secret lifecycle integrated with Azure Key Vault.

### Modified Capabilities
- None.

## Impact

- Affects `databases/data/warehouse/` and `databases/data/warehouse-dev/` CNPG cluster manifests and related role configuration.
- Adds or updates manifests for a reconciliation workload (Job/CronJob or equivalent orchestration) and associated SQL/config assets.
- Affects consumer namespaces that require warehouse read access (at minimum `airflow` and `streamlit`).
- Introduces or updates ExternalSecret/SecretStore usage for admin LOGIN credentials backed by Azure Key Vault.
- Changes operational runbooks for warehouse access verification and bootstrap sequencing.
