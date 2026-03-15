## Why

The Streamlit sales app is deployed and reachable, but runtime requests fail with a backend configuration error because `SALES_WAREHOUSE_URL` is rendered with `postgresql://` instead of the required `postgresql+psycopg://` scheme. We need a spec-backed fix now so the app can connect with psycopg v3 and so future deployments validate this contract before rollout.

## What Changes

- Update the production Streamlit secret templating contract to render `SALES_WAREHOUSE_URL` with the `postgresql+psycopg://` driver prefix.
- Define pre-rollout and post-rollout validation requirements that verify effective environment configuration and app-level behavior (not only pod readiness).
- Capture cross-repo triage handoff requirements for runtime errors discovered in homelab so upstream app maintainers receive actionable evidence (traceback, image digest, runtime command context).

## Capabilities

### New Capabilities
- `streamlit-sales-runtime-configuration`: Declarative requirements for Streamlit sales app database URL construction, runtime validation, and operational error handoff.

### Modified Capabilities
- None.

## Impact

- Affected code: `apps/production/sales-pipeline-pulse/externalsecret.yaml` and associated runbook/spec artifacts for validation.
- Affected systems: External Secrets templating, Streamlit app runtime configuration, warehouse connectivity from `streamlit` namespace.
- Operational impact: rollout checks must include app-level functional validation and documented upstream handoff artifacts when runtime defects are observed.
