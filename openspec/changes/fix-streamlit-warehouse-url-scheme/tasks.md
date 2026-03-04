## 1. Runtime Configuration Contract Fix

- [x] 1.1 Update `apps/production/sales-pipeline-pulse/externalsecret.yaml` so `SALES_WAREHOUSE_URL` uses `postgresql+psycopg://`.
- [x] 1.2 Render/validate manifests to confirm the ExternalSecret template remains syntactically valid after the DSN scheme change.

## 2. Rollout Validation Hardening

- [x] 2.1 Add or update runbook/spec notes to require request-path validation and runtime log review for Streamlit rollouts.
- [x] 2.2 Execute validation in cluster by confirming rendered secret prefix and exercising at least one app request path.

## 3. Upstream Handoff Workflow

- [x] 3.1 Create a standardized handoff checklist for app-side failures (traceback, image tag/digest, container command, working directory).
- [x] 3.2 Capture and share current runtime evidence with the Streamlit repo owner if post-fix request-path errors remain.
