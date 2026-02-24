## 1. Cross-Namespace RBAC and SecretStore

- [x] 1.1 Create `apps/production/airflow/warehouse-access.yaml` containing: Role `external-secrets-warehouse-reader` in `warehouse` namespace, RoleBinding `airflow-external-secrets-warehouse-reader` in `warehouse` namespace binding the `external-secrets-kubernetes-reader` SA from `airflow`, and SecretStore `kubernetes-warehouse` in `airflow` namespace with `remoteNamespace: warehouse`
- [x] 1.2 Add `warehouse-access.yaml` to `apps/production/airflow/kustomization.yaml` resources list

## 2. Update Warehouse Connection ExternalSecret

- [x] 2.1 Rewrite `apps/production/airflow/warehouse-postgres-conn-externalsecret.yaml` to use `kubernetes-warehouse` SecretStore, reading `uri` property from `warehouse-db-production-cnpg-v1-app` secret, with `refreshInterval: 30s` and no template section

## 3. Verification

- [ ] 3.1 Confirm `kubernetes-warehouse` SecretStore shows `Ready: True` in `airflow` namespace
- [ ] 3.2 Confirm `warehouse-postgres-conn` ExternalSecret syncs successfully and K8s Secret exists with valid PostgreSQL URI in `AIRFLOW_CONN_WAREHOUSE_POSTGRES` key
- [ ] 3.3 Confirm `postgres_ping` DAG succeeds using `warehouse_postgres` conn_id
