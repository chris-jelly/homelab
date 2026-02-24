## Context

The Airflow deployment needs a `warehouse-postgres-conn` K8s Secret containing `AIRFLOW_CONN_WAREHOUSE_POSTGRES` (a PostgreSQL URI). The warehouse database is a CNPG cluster in the `warehouse` namespace, while Airflow runs in the `airflow` namespace.

CNPG automatically generates a secret (`warehouse-db-production-cnpg-v1-app`) containing a complete `uri` field: `postgresql://app:<password>@warehouse-db-production-cnpg-v1-rw.warehouse:5432/app`. This URI already contains everything Airflow needs.

The current implementation (from `airflow3-dag-secrets` change) attempts to fetch the password from Azure KV and compose the URI via an ESO template. This fails because the KV secret was never provisioned. The Airflow metadata database already uses a working pattern: an ESO Kubernetes provider that reads the CNPG secret directly from the same namespace. The warehouse connection needs the same approach, extended to work cross-namespace.

The existing infrastructure in `apps/production/airflow/database-externalsecret.yaml` provides a model:
- `external-secrets-kubernetes-reader` ServiceAccount in `airflow`
- `kubernetes-local` SecretStore pointing at `remoteNamespace: airflow`
- Role + RoleBinding scoping secret read access
- ExternalSecret mapping CNPG `uri` field to an Airflow-consumable key

## Goals / Non-Goals

**Goals:**
- Source the warehouse PostgreSQL connection URI directly from the CNPG-generated secret, eliminating the Azure KV dependency
- Enable cross-namespace secret access from `airflow` to `warehouse` via ESO Kubernetes provider
- Maintain the same output secret name and key (`warehouse-postgres-conn` / `AIRFLOW_CONN_WAREHOUSE_POSTGRES`) so DAGs require no changes
- Make the pattern reusable for future consumers of the warehouse database

**Non-Goals:**
- Modifying the Salesforce connection -- it correctly uses Azure KV since those credentials are external to the cluster
- Changing the warehouse CNPG cluster configuration or namespace
- Provisioning warehouse access for other namespaces (streamlit, etc.) -- that's future work using the same RBAC pattern
- Removing `app--warehouse-postgres-password--prod` from Azure KV (it doesn't exist anyway; no cleanup needed)

## Decisions

### 1. Reuse existing ServiceAccount, add cross-namespace RBAC

The `external-secrets-kubernetes-reader` ServiceAccount already exists in the `airflow` namespace. Rather than creating a new SA, we add a Role in `warehouse` granting secret read access and a RoleBinding binding the existing airflow SA to it.

**RBAC structure:**

```yaml
# In warehouse namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: external-secrets-warehouse-reader
  namespace: warehouse
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["authorization.k8s.io"]
  resources: ["selfsubjectrulesreviews"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: airflow-external-secrets-warehouse-reader
  namespace: warehouse
subjects:
- kind: ServiceAccount
  name: external-secrets-kubernetes-reader
  namespace: airflow
roleRef:
  kind: Role
  name: external-secrets-warehouse-reader
  apiGroup: rbac.authorization.k8s.io
```

The RoleBinding name includes `airflow-` prefix to indicate which consumer it serves. Future consumers (e.g., streamlit) would add their own RoleBinding to the same Role.

**Alternative considered:** Create a ClusterRole for cross-namespace secret reading. Rejected because it grants broader access than needed -- a namespaced Role in `warehouse` follows least privilege.

### 2. New SecretStore pointing at warehouse namespace

A second SecretStore in the `airflow` namespace, `kubernetes-warehouse`, points at `remoteNamespace: warehouse`. This is separate from `kubernetes-local` (which points at `airflow` itself) because each SecretStore can only target one namespace.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: kubernetes-warehouse
  namespace: airflow
spec:
  provider:
    kubernetes:
      remoteNamespace: warehouse
      server:
        caProvider:
          type: ConfigMap
          name: kube-root-ca.crt
          key: ca.crt
      auth:
        serviceAccount:
          name: external-secrets-kubernetes-reader
```

### 3. Simplified ExternalSecret -- key rename only

The CNPG secret's `uri` field contains the complete PostgreSQL connection URI. The ExternalSecret becomes a simple key rename, no template composition needed:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: warehouse-postgres-conn
  namespace: airflow
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: kubernetes-warehouse
    kind: SecretStore
  target:
    name: warehouse-postgres-conn
    creationPolicy: Owner
  data:
    - secretKey: AIRFLOW_CONN_WAREHOUSE_POSTGRES
      remoteRef:
        key: warehouse-db-production-cnpg-v1-app
        property: uri
```

Using `refreshInterval: 30s` to match the metadata connection ExternalSecret pattern (faster sync than the 1h used for Azure KV sources, since in-cluster reads are cheap).

**Alternative considered:** Keep the ESO template approach and compose the URI from individual CNPG secret fields (password, host, etc.). Rejected because the CNPG `uri` field already contains the exact format needed -- composition adds complexity and divergence risk.

### 4. Consumer-owned RBAC placement

The RBAC resources (Role + RoleBinding in `warehouse` namespace) are defined in the airflow app directory (`apps/production/airflow/`), not in `databases/data/warehouse/`. This follows the "consumer declares its dependencies" pattern -- the airflow deployment owns all the resources it needs, including cross-namespace access grants.

A new file `apps/production/airflow/warehouse-access.yaml` contains the Role, RoleBinding, and SecretStore as a multi-document YAML (mirroring the pattern in `database-externalsecret.yaml`).

**Alternative considered:** Place RBAC in `databases/data/warehouse/`. Rejected because the warehouse doesn't need to know about its consumers -- each consumer declares what access it needs, and the RoleBinding makes this auditable via `kubectl get rolebindings -n warehouse`.

### 5. CNPG URI host format

The CNPG-generated URI uses the short service name: `warehouse-db-production-cnpg-v1-rw.warehouse:5432`. This resolves correctly from the `airflow` namespace via cluster DNS (expanded to `warehouse-db-production-cnpg-v1-rw.warehouse.svc.cluster.local`). No modification of the URI is needed.

## Risks / Trade-offs

**[CNPG secret name is hardcoded in ExternalSecret]** The `remoteRef.key` references `warehouse-db-production-cnpg-v1-app` directly. If the CNPG cluster is renamed or recreated with a different name, the ExternalSecret breaks.
→ Mitigation: The CNPG cluster name is stable and defined in `databases/data/warehouse/database.yaml`. Changes would be a major operation requiring coordination anyway.

**[Cross-namespace RBAC increases attack surface]** The `external-secrets-kubernetes-reader` SA in `airflow` can now read all secrets in `warehouse`. If the SA token is compromised, warehouse secrets are exposed.
→ Mitigation: The SA is only used by ESO's controller (not mounted in application pods). The Role is scoped to secrets only (no other resources). This is the same trust model as the existing `kubernetes-local` pattern.

**[Refresh interval difference from Salesforce connection]** The warehouse ExternalSecret uses `30s` while the Salesforce one uses `1h`. This is intentional (in-cluster reads are cheap, Azure KV has rate limits) but may cause brief confusion if someone compares them.
→ Mitigation: Document the rationale in the manifest comments.
