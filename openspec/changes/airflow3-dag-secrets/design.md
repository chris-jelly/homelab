## Context

Airflow 3 DAGs in `de-airflow-pipeline` use `conn_id`-based hooks and self-inject connection secrets via KubernetesExecutor `executor_config` pod overrides. Each task's pod override references a named K8s Secret and key:

| K8s Secret Name | Secret Key | Format | Used By |
|---|---|---|---|
| `warehouse-postgres-conn` | `AIRFLOW_CONN_WAREHOUSE_POSTGRES` | PostgreSQL URI | `postgres_ping`, `salesforce_extraction` |
| `salesforce-conn` | `AIRFLOW_CONN_SALESFORCE` | JSON blob | `salesforce_extraction` |

These K8s Secrets do not exist yet. The homelab repo currently provisions `postgres-credentials` and `salesforce-credentials` with individual key-value fields (host, database, username, password / consumer_key, consumer_secret) that no DAG references.

The existing `postgres-credentials` ExternalSecret already uses ESO templating to hardcode connection details and inject a password from Azure KV. The new ExternalSecrets follow the same pattern but compose a single connection URI/JSON instead of individual fields.

## Goals / Non-Goals

**Goals:**
- Provision `warehouse-postgres-conn` and `salesforce-conn` K8s Secrets in the `airflow` namespace via ExternalSecrets Operator
- Use ESO templates to compose Airflow 3 connection URIs/JSON from Azure KV source secrets
- Remove obsolete `postgres-credentials` and `salesforce-credentials` ExternalSecrets
- Update the production Kustomization resource list

**Non-Goals:**
- Helm release changes -- DAGs handle secret injection via pod overrides, no `extraEnvFrom` modification needed
- Airflow secrets backend configuration -- evaluated and rejected (two-vault problem, loss of pod isolation, manual URI composition)
- Salesforce auth flow changes -- this change provisions whatever fields the Airflow Salesforce provider needs; determining the exact fields is a prerequisite, not a design decision
- DAG repo changes -- the conn_id naming fix is handled separately in `de-airflow-pipeline`

## Decisions

### 1. ESO templates for URI composition (not pre-composed KV secrets)

The warehouse PostgreSQL URI includes a password from Azure KV and a host that is a deterministic CNPG service name. Rather than manually composing and storing the full URI in Azure KV (which would require manual updates on password rotation), the ExternalSecret template composes the URI at sync time.

**warehouse-postgres-conn template structure:**

```yaml
spec:
  secretStoreRef:
    name: azure-kv-store
    kind: ClusterSecretStore
  target:
    name: warehouse-postgres-conn
    creationPolicy: Owner
    template:
      data:
        AIRFLOW_CONN_WAREHOUSE_POSTGRES: "postgresql://app:{{ .password }}@warehouse-db-production-cnpg-v1-rw.warehouse.svc.cluster.local:5432/app"
  data:
  - secretKey: password
    remoteRef:
      key: app--warehouse-postgres-password--prod
```

This mirrors the existing `postgres-credentials` pattern but outputs a single composed URI instead of individual fields.

**Alternative considered:** Store the full URI directly in Azure KV. Rejected because the password would need to be embedded in the URI value in KV, making rotation a manual multi-step process (update KV, ensure URI format is correct). ESO templating keeps the password as the single source of truth.

### 2. Salesforce connection uses `azure-kv-work-integrations-store`

The Salesforce credentials live in a separate Azure Key Vault (`kv-work-integrations`), accessed via the `azure-kv-work-integrations-store` ClusterSecretStore. The new `salesforce-conn` ExternalSecret must use this same store -- the credentials cannot be moved to the primary vault without re-provisioning.

**salesforce-conn template structure:**

```yaml
spec:
  secretStoreRef:
    name: azure-kv-work-integrations-store
    kind: ClusterSecretStore
  target:
    name: salesforce-conn
    creationPolicy: Owner
    template:
      data:
        AIRFLOW_CONN_SALESFORCE: |
          {"conn_type": "salesforce", "login": "airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com", "extra": {"consumer_key": "{{ .consumer_key }}", "private_key": "{{ .private_key }}", "domain": "login"}}
  data:
  - secretKey: consumer_key
    remoteRef:
      key: app--salesforce-consumer-key--prod
  - secretKey: private_key
    remoteRef:
      key: app--salesforce-private-key--prod
```

**RESOLVED -- Salesforce auth flow determined.** Research into the Airflow Salesforce provider docs and the Salesforce org configuration confirmed the following:

1. **Auth flow**: OAuth 2.0 JWT Bearer. This is the only OAuth flow the Airflow `SalesforceHook` supports. The "integrations" External Client App already has `OptionsAllowAdminApprovedUsersOnly: true`, which is the correct policy for JWT bearer.
2. **Required fields**: The Airflow provider needs `consumer_key`, `private_key`, and `domain` in the connection `extra` dict, plus `login` (username) at the connection level. `consumer_secret` is NOT used in JWT bearer -- the private key signs the JWT assertion instead.
3. **New Azure KV secret required**: `app--salesforce-private-key--prod` (RSA private key, 2048-bit). The existing `app--salesforce-consumer-secret--prod` is no longer needed for this flow but can remain in KV.

**Salesforce org prerequisites** (manual, outside GitOps):
- Generate RSA key pair (2048-bit, X.509 cert, 365-day validity)
- Upload public certificate to "integrations" Connected App (via Setup > App Manager > integrations > Edit, not the External Client App UI which doesn't expose the certificate upload field)
- Create integration user: `Salesforce` license (full), `Read Only` profile, username `airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com`
- Populate `Integration_App_Access` permission set with ViewAll on: Account, Contact, Opportunity, Lead, Case, Campaign; Read on: Product2, Pricebook2 (ViewAll not permitted on these objects). OpportunityHistory, PricebookEntry, and User are not directly permissionable (OpportunityHistory inherits from Opportunity, PricebookEntry from Pricebook2, User is readable by all internal users).
- Assign permission set to integration user (pre-authorizes for Connected App)
- Store RSA private key in Azure KV as `app--salesforce-private-key--prod`
- Verify with: `sf org login jwt --client-id <consumer_key> --jwt-key-file <key> --username airflow-integration@orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com --instance-url https://orgfarm-d58c800d67-dev-ed.develop.my.salesforce.com`

**License change from original plan:** The `Salesforce Integration` license was initially chosen for least-privilege API-only access. However, this license does not permit access to standard CRM objects (Account, Contact, Opportunity, Lead, Case, Campaign). Since the extraction DAG needs to read these objects, the integration user was switched to a full `Salesforce` license with the `Read Only` profile. This consumes 1 of the org's 4 full Salesforce licenses (2 were previously used by admin accounts, leaving 1 remaining after this change). The `Read Only` profile provides base read access to all standard objects; the `Integration_App_Access` permission set adds ViewAll where permitted to ensure org-wide visibility regardless of record ownership.

**Alternative considered:** Airflow AzureKeyVaultBackend as a native secrets backend. Rejected because: (1) it accepts a single `vault_url` and cannot span two vaults, (2) it gives the scheduler and API server access to all connection credentials (violating pod-level isolation), (3) connection URIs must be pre-composed and stored manually in KV, (4) connections move outside GitOps visibility.

### 3. No Helm release changes required

DAGs declare their secret dependencies in `executor_config` pod overrides using typed `kubernetes.client.models` objects. Each task specifies exactly which K8s Secrets to mount as env vars. This means:

- The scheduler and API server never receive data credentials
- Each DAG/task gets only the secrets it declares
- Adding a new connection to a DAG requires only: (a) a new ExternalSecret in the homelab repo, (b) a pod override in the DAG code

No `extraEnvFrom`, `workers.extraEnvFrom`, or pod template changes are needed in the Helm release.

### 4. Atomic switchover via single Kustomization update

The old and new ExternalSecrets produce different K8s Secret names (`postgres-credentials` vs `warehouse-postgres-conn`). This means both can coexist during rollout. The switchover sequence:

1. Add new ExternalSecret manifests and Kustomization entries
2. Flux reconciles -- new K8s Secrets are created alongside old ones
3. DAGs (already updated in `de-airflow-pipeline`) reference the new secret names
4. Verify DAGs work (`postgres_ping` hourly, `salesforce_extraction` daily)
5. Remove old ExternalSecret manifests and Kustomization entries in a follow-up commit

This avoids a big-bang cutover. However, if preferred, steps 1-2 and 4-5 can be combined into a single commit since the old secrets are already unreferenced by the current DAGs.

## Risks / Trade-offs

**[Salesforce connection format -- RESOLVED]** The auth flow is JWT Bearer. Required fields: `consumer_key`, `private_key`, `domain` (in extra), plus `login` (username) at the connection level. `consumer_secret` is not used. A new Azure KV secret (`app--salesforce-private-key--prod`) and Salesforce org setup (integration user, certificate upload, permission set) are prerequisites.

**[CNPG password not in sync with KV]** The warehouse database password in Azure KV (`app--warehouse-postgres-password--prod`) was manually stored. If the CNPG operator rotates the `app` user password, the KV secret becomes stale and the connection URI will use the wrong password.
-> Mitigation: This is a pre-existing risk (the current `postgres-credentials` ExternalSecret has the same dependency). Out of scope for this change, but worth noting for future work.

**[Old secrets removed while still consumed]** The proposal marks removal of `postgres-credentials` and `salesforce-credentials` as BREAKING. If any other workload reads these secrets, removing them will break that workload.
-> Mitigation: Grep the repo for references to these secret names before removal. Currently no other consumer is known.

**[DAG repo must be updated first]** The new K8s Secrets are only useful once DAGs reference them via pod overrides. If the homelab changes deploy before the DAG repo updates, the new secrets exist but are unused (harmless). If the old secrets are removed before DAGs stop referencing them, DAGs break.
-> Mitigation: The DAG repo changes are already in progress. Deploy new secrets first, verify, then remove old ones.
