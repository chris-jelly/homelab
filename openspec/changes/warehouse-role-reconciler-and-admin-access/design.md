## Context

Warehouse database objects (schemas, tables, views, sequences) are created by DAG workflows after the cluster is bootstrapped. A one-time SQL grant at bootstrap cannot reliably cover objects that do not yet exist, which causes read-only access drift for downstream consumers.

The repository already uses CloudNative-PG (CNPG), Flux GitOps, and External Secrets Operator. CNPG can declaratively reconcile role existence and passwords, but object-level privileges still require SQL grants/default privileges. We also need a human pgAdmin login for warehouse operations, with credentials managed through Azure Key Vault rather than hardcoded Kubernetes secrets.

## Goals / Non-Goals

**Goals:**
- Ensure warehouse read access for Airflow and Streamlit converges automatically as DAGs create or change objects.
- Split identity management (roles/passwords) from privilege management (grants/default privileges) with clear ownership.
- Provide an admin LOGIN role for pgAdmin backed by Azure Key Vault secret material and GitOps manifests.
- Keep behavior idempotent, repeatable, and observable in a fresh-cluster bootstrap and ongoing operations.

**Non-Goals:**
- Redesign DAG ownership or full warehouse schema architecture.
- Introduce ad hoc manual SQL as the primary access model.
- Grant blanket superuser rights to application roles.
- Replace existing CNPG/Flux/ESO platform components.

## Decisions

### 1) Two-plane access model
Use two distinct reconciliation planes:
- **Role plane (CNPG managed roles):** source of truth for role existence, login capability, role membership, and passwordSecret references.
- **Privilege plane (SQL reconciler):** source of truth for grants on existing objects and default privileges for future objects.

Rationale:
- CNPG is strong for role lifecycle drift correction.
- SQL reconciler is required for object-level privileges that depend on runtime object creation.

Alternative considered:
- **Single bootstrap SQL Job only:** rejected because it cannot guarantee convergence after new DAG-created objects appear.

### 2) Convergent privilege reconciler with periodic execution
Define an idempotent reconciler workload that executes SQL repeatedly (for example: on deploy and periodically via CronJob). The reconciler SHALL:
- grant database CONNECT to reader principals,
- grant schema USAGE for permitted schemas,
- grant SELECT on all existing tables/views in permitted schemas,
- grant sequence privileges when readers need sequence-backed introspection,
- apply ALTER DEFAULT PRIVILEGES for future objects.

Rationale:
- Handles both fresh bootstrap and continuous DAG evolution.
- Safe reruns provide self-healing for privilege drift.

Alternative considered:
- **Only ALTER DEFAULT PRIVILEGES:** rejected because defaults do not retroactively grant existing objects.

### 3) Single writer-role strategy for default privileges (Option A)
Standardize object creation through one DAG writer role per environment for warehouse DDL-producing workloads. Default privileges are then managed for this single writer role.

Rationale:
- Greatly simplifies privilege correctness because Postgres default privileges are scoped to object creator role.
- Reduces combinatorial complexity from many creator roles.

Alternative considered:
- **Multiple writer roles:** possible, but significantly higher maintenance and drift risk; would require reconciler loops per creator role.

### 4) Consumer-specific read principals including Streamlit
Define consumer login roles (or login users mapped to a shared read group role) for:
- Airflow warehouse reads
- Streamlit warehouse reads

Both consumers SHALL receive equivalent read-only capability for the approved schema set through shared reconciliation logic.

Rationale:
- Explicitly encodes Streamlit as a first-class consumer in access policy.

Alternative considered:
- **Single shared login credential across consumers:** rejected for auditability and blast-radius reasons.

### 5) Admin LOGIN role uses Key Vault-backed password lifecycle
Add a dedicated admin LOGIN role for pgAdmin operations with least-privilege role attributes by default (non-superuser unless explicitly required by a separate decision). The admin password SHALL be sourced from Azure Key Vault via External Secrets into Kubernetes, then referenced by CNPG `passwordSecret`.

Rationale:
- Reconciler is the right mechanism for object privileges, not for password custody.
- Key Vault provides central secret rotation and custody controls.

Alternative considered:
- **Reconciler sets admin password directly in SQL:** rejected due to weaker secret management posture and less alignment with existing ESO + Key Vault patterns.

## Risks / Trade-offs

- **[Risk] Default privileges miss objects created by unexpected roles** -> Mitigation: enforce single writer role contract and add reconciler checks for owner-role drift.
- **[Risk] Reconciler SQL failure blocks convergence** -> Mitigation: idempotent statements, retry schedule, and alerting on repeated failures.
- **[Risk] Over-broad admin role** -> Mitigation: start with least privilege, separate break-glass elevation path, and document explicit escalation procedure.
- **[Risk] Schema scope creep** -> Mitigation: keep explicit allowlist of schemas in reconciler config and review via PR.
- **[Risk] Secret rotation timing mismatch** -> Mitigation: align ESO refresh interval with operational rotation policy and verify CNPG reconciliation behavior during credential rollover.

## Migration Plan

1. Introduce managed roles and secret references for read consumers and admin login in warehouse and warehouse-dev manifests.
2. Deploy reconciler workload with no-op-safe SQL and explicit schema allowlist.
3. Validate convergence on existing objects (Airflow + Streamlit can read approved schemas).
4. Validate future-object behavior by creating new DAG-managed objects and confirming read access without manual grants.
5. Roll out pgAdmin admin login secret flow from Key Vault, then validate login/connectivity and role boundaries.
6. Document runbook checks for drift detection, reconciler health, and bootstrap sequencing.

Rollback:
- Disable reconciler workload and restore previous static grants if needed.
- Revert role manifest changes via GitOps and let CNPG reconcile back to prior state.

## Open Questions

- Should sequence privileges be granted universally or only when reader queries require direct sequence access?
- Should admin login be environment-scoped (`warehouse-admin-prod`, `warehouse-admin-dev`) or one role name with environment-specific secret/materialization?
- Is PostgreSQL 13 -> 15 migration for warehouse environments coupled to this change or tracked as a dedicated follow-up change?
