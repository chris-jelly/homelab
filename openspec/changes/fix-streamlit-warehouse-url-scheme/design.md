## Context

The `sales-pipeline-pulse` app is deployed in the `streamlit` namespace and receives `SALES_WAREHOUSE_URL` through an ExternalSecret template. Runtime errors now confirm a contract mismatch: the app requires `postgresql+psycopg://` (psycopg v3), while the current manifest renders `postgresql://`.

Current operating constraints:
- Deployments are Flux-managed from this repository.
- Secret values must remain managed through External Secrets and existing cross-namespace SecretStore patterns.
- Pod readiness alone is insufficient to prove app correctness because Streamlit can serve health endpoints while request-time app execution fails.

Stakeholders:
- Homelab platform owner (deployment and operational validation)
- Streamlit app repository owner (application runtime behavior and packaging fixes)

## Goals / Non-Goals

**Goals:**
- Enforce the DSN driver prefix contract for `SALES_WAREHOUSE_URL` in homelab manifests.
- Define validation requirements that assert effective runtime behavior, not only Kubernetes health status.
- Define a lightweight cross-repo handoff contract when homelab discovers app runtime failures.

**Non-Goals:**
- Redesigning Streamlit application internals.
- Changing warehouse role naming, grants, or reconciliation logic.
- Introducing a new secret backend or replacing External Secrets.

## Decisions

### Decision: Correct DSN scheme at secret-template source
- Rationale: The DSN is synthesized in `apps/production/sales-pipeline-pulse/externalsecret.yaml`, so correcting the scheme there fixes the rendered secret without changing app code.
- Alternatives considered:
  - Patch app code to accept `postgresql://`: rejected because the app explicitly requires psycopg v3 driver semantics and this weakens contract clarity.
  - Inject a second environment variable for translated DSN: rejected as unnecessary complexity.

### Decision: Add app-level rollout validation criteria
- Rationale: Readiness/liveness probes can pass while Streamlit session execution fails; rollout checks must include app-path validation and error-log review.
- Alternatives considered:
  - Keep probe-only validation: rejected because it misses currently observed failure mode.

### Decision: Capture upstream handoff evidence as an operational requirement
- Rationale: Some defects are in the application repository and cannot be solved in homelab manifests; preserving a standard evidence packet accelerates upstream debugging and prevents repeated triage.
- Alternatives considered:
  - Ad hoc handoff per incident: rejected due to inconsistency and slower resolution.

## Risks / Trade-offs

- [DSN scheme updated but another runtime defect remains] -> Require explicit post-deploy app-path validation and include fallback escalation to upstream repo.
- [Validation burden increases operational effort] -> Keep required evidence minimal and reusable (traceback, image digest, effective runtime command).
- [Cross-repo ownership ambiguity] -> Explicitly document what homelab owns (wiring/secret rendering) vs what upstream owns (app runtime logic).

## Migration Plan

1. Update the Streamlit ExternalSecret template to render `postgresql+psycopg://`.
2. Let Flux reconcile and verify the generated Kubernetes secret in `streamlit` namespace reflects the new scheme.
3. Validate app behavior through request-path checks and logs, not only probe status.
4. If runtime errors persist, collect standardized evidence and hand off to the Streamlit repository owner for app-side remediation.

Rollback:
- Revert the manifest change in Git if unintended side effects are detected.
- Reconcile through Flux and confirm previous secret rendering is restored.

## Open Questions

- Should homelab add an automated synthetic check for Streamlit page execution to catch request-path failures earlier than manual validation?
