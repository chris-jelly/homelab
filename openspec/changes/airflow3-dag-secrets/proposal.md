## Why

The `de-airflow-pipeline` repo has been migrated to Airflow 3. DAGs now use `conn_id`-based hooks (e.g. `PostgresHook(conn_id="warehouse_postgres")`) instead of reading individual credential environment variables. Airflow 3 expects connections to be provided as pre-composed `AIRFLOW_CONN_*` environment variables — a single URI or JSON blob per connection — mounted into executor pods. The existing ExternalSecrets (`postgres-credentials`, `salesforce-credentials`) emit individual keys that no DAG references anymore.

DAGs self-inject their required secrets via KubernetesExecutor `executor_config` pod overrides (ADR-002 pod-level secret isolation). Each task declares exactly which K8s secrets it needs, and only those task pods receive them. The scheduler, API server, and unrelated DAGs never see data credentials. This means the homelab repo only needs to provision the K8s secrets — no Helm values changes are required.

An Airflow-native Azure Key Vault secrets backend was evaluated and rejected for this scope. The two-vault problem (Salesforce credentials live in `kv-work-integrations`, not the primary vault), loss of pod-level isolation (backend gives all Airflow components access to all connections), manual URI composition (no ESO templating), and reduced GitOps visibility made it a poor fit for two connections.

## What Changes

- **Add** ExternalSecret `warehouse-postgres-conn` in the `airflow` namespace producing a single `AIRFLOW_CONN_WAREHOUSE_POSTGRES` key containing a PostgreSQL connection URI. The URI must be composed from existing secrets (host from CNPG service, password from Azure KV) since Kubernetes doesn't support env var interpolation. Use an ESO template to assemble the URI.
- **Add** ExternalSecret `salesforce-conn` in the `airflow` namespace producing a single `AIRFLOW_CONN_SALESFORCE` key containing a Salesforce connection JSON blob. Compose from existing individual secrets in Azure KV (`azure-kv-work-integrations-store`) via ESO template. The Salesforce connection uses OAuth 2.0 with `consumer_key` and `consumer_secret` (both already in KV). The exact connection JSON format (whether a `refresh_token` or other fields are needed beyond `consumer_key`/`consumer_secret`/`domain`) requires validation against the Airflow Salesforce provider during implementation.
- **Remove** ExternalSecret `salesforce-credentials` (keys: `consumer_key`, `consumer_secret`) — no longer referenced by any DAG. **BREAKING** if any other consumer reads this secret.
- **Remove** ExternalSecret `postgres-credentials` (keys: `host`, `database`, `username`, `password`) — no longer referenced by any DAG. **BREAKING** if any other consumer reads this secret.

## Capabilities

### New Capabilities
- `airflow-dag-connections`: Provisioning of Airflow 3 `AIRFLOW_CONN_*` connection secrets as K8s Secrets in the `airflow` namespace, using ESO templates to compose connection URIs/JSON from Azure Key Vault source secrets.

### Modified Capabilities
- `airflow-deployment`: Old credential ExternalSecrets (`postgres-credentials`, `salesforce-credentials`) and their corresponding resource entries in the Kustomization must be removed.

## Impact

- **ExternalSecrets**: Two new ExternalSecret manifests added; two existing ones removed. The `postgres-credentials` and `salesforce-credentials` K8s secrets will be deleted once their ExternalSecret owners are removed.
- **Azure Key Vault**: No new KV secrets required for the warehouse connection — the password is already in KV (`app--warehouse-postgres-password--prod`). Salesforce credentials (`consumer_key`, `consumer_secret`) are already in a separate vault (`azure-kv-work-integrations-store`). May need to verify whether additional fields (e.g. `refresh_token`) are required for the Airflow 3 Salesforce connection format.
- **Helm values**: No changes required. DAGs self-inject secrets via `executor_config` pod overrides — the Helm release does not need `extraEnvFrom` or worker-specific env configuration for data connections.
- **Kustomization**: `apps/production/airflow/kustomization.yaml` needs resource list updated (add new, remove old ExternalSecret files).
- **CNPG interaction**: The warehouse database URI must reference `warehouse-db-production-cnpg-v1-rw.warehouse.svc.cluster.local:5432` — this host is currently hardcoded in the `postgres-credentials` ESO template and will be hardcoded in the new template. The ESO template composes the full URI, so the scheme (`postgresql://`) and any query parameters are under our control.
- **DAG repo coordination**: The `de-airflow-pipeline` repo has a separate fix in progress to remove vestigial environment-suffix logic from conn_ids (e.g. `warehouse_postgres_dev` becoming `warehouse_postgres`). The K8s secret names and keys in this change align with the target conn_ids after that fix.
- **DAG verification**: `postgres_ping` (hourly) and `salesforce_extraction` (daily) DAGs will validate the full chain once deployed.
