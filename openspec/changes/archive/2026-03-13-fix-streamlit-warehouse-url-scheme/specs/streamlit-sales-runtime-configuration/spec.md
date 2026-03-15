## ADDED Requirements

### Requirement: Streamlit warehouse URL SHALL use psycopg v3 DSN scheme
The production `sales-pipeline-pulse` runtime configuration SHALL render `SALES_WAREHOUSE_URL` with the `postgresql+psycopg://` scheme when composing the warehouse connection string from Kubernetes-managed secret material.

#### Scenario: ExternalSecret template produces psycopg URL
- **WHEN** `apps/production/sales-pipeline-pulse/externalsecret.yaml` is rendered and synced into `sales-pipeline-pulse-secrets`
- **THEN** the effective `SALES_WAREHOUSE_URL` value begins with `postgresql+psycopg://`

### Requirement: Streamlit rollout validation SHALL include request-path behavior
Operational validation for `sales-pipeline-pulse` SHALL include at least one request-path verification and log inspection step so app execution errors are detected even when Kubernetes health probes report Ready.

#### Scenario: Probe-only success does not satisfy validation
- **WHEN** the pod reports healthy readiness/liveness probe status
- **THEN** rollout validation still requires a request-path check and recent runtime log review before declaring success

### Requirement: Runtime defect handoff SHALL provide standardized evidence
When homelab detects a runtime defect that likely belongs in the Streamlit application repository, operators SHALL provide a standardized evidence packet to upstream maintainers including traceback details, deployed image reference, and effective container startup context.

#### Scenario: Upstream handoff packet is generated for app-side failures
- **WHEN** request-path validation surfaces an application runtime exception after deployment
- **THEN** the handoff includes the observed traceback, deployed image tag or digest, and container command/working-directory evidence collected from the cluster
