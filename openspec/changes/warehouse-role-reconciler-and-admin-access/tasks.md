## 1. Role and secret foundations

- [x] 1.1 Add CNPG managed role definitions for warehouse read consumers (Airflow and Streamlit) in the production warehouse cluster manifest.
- [x] 1.2 Add CNPG managed role definition for warehouse admin LOGIN role in production with explicitly declared privilege posture.
- [x] 1.3 Add/update ExternalSecret resources so admin LOGIN password material is sourced from Azure Key Vault into Kubernetes secrets consumed by CNPG `passwordSecret`.
- [x] 1.4 Document role naming and environment mapping contract (prod/dev) for read and admin principals.

## 2. Privilege reconciler implementation

- [x] 2.1 Add reconciler workload manifests (on-deploy execution plus periodic convergence) for warehouse.
- [x] 2.2 Implement idempotent SQL for CONNECT, schema USAGE, and SELECT grants on existing objects for approved schemas.
- [x] 2.3 Implement ALTER DEFAULT PRIVILEGES SQL for the designated writer role so future DAG-created objects inherit read access.
- [x] 2.4 Add explicit Streamlit coverage in reconciler configuration and SQL grant targets alongside Airflow.
- [x] 2.5 Add configuration validation and logging expectations to make reconciliation failures observable.

## 3. Verification and operations

- [ ] 3.1 Validate from-scratch bootstrap flow: roles reconcile before object creation, reconciler no-ops safely, then converges grants after DAG object creation.
- [ ] 3.2 Validate Airflow read access to existing and newly created approved-schema objects.
- [ ] 3.3 Validate Streamlit read access to existing and newly created approved-schema objects.
- [ ] 3.4 Validate pgAdmin admin LOGIN can connect with Key Vault-backed credentials and enforce the configured privilege posture.
- [x] 3.5 Update runbook/spec docs with troubleshooting, drift-detection checks, and rollback notes.
