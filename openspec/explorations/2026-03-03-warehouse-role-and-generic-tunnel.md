# Exploration: Warehouse Read-Only Role and Generic Cloudflare Tunnel

Date: 2026-03-03

## Context

We explored two related topics:

1. How to provision a missing read-only warehouse database user declaratively (instead of manual SSH/psql steps).
2. How to make Cloudflare tunnel routing generic so Streamlit can be exposed cleanly, potentially replacing Linkding routing.

This note captures the current understanding and recommended direction in explore mode (no implementation yet).

## What We Verified in This Repo

- Warehouse PostgreSQL version is explicitly pinned to PG13 in YAML:
  - `databases/data/warehouse/database.yaml` (`imageName: ghcr.io/cloudnative-pg/postgresql:13`)
  - `databases/data/warehouse-dev/database.yaml` (`imageName: ghcr.io/cloudnative-pg/postgresql:13`)
- Airflow database is already PG15:
  - `databases/data/airflow/database.yaml` (`imageName: ghcr.io/cloudnative-pg/postgresql:15`)
- Repo standard says PG15 should be used:
  - `specs/003-development-standards.md` (PostgreSQL 15 guideline)
- Existing Cloudflare tunnel pattern is app-local (not generic), e.g. Linkding:
  - `apps/production/linkding/cloudflare.yaml`
- Existing cross-namespace External Secrets access pattern exists for Airflow reading warehouse CNPG secret:
  - `apps/production/airflow/warehouse-access.yaml`
  - `apps/production/airflow/warehouse-postgres-conn-externalsecret.yaml`

## Declarative User Provisioning: Preferred Direction

### Recommendation

Use CloudNativePG declarative role management (`.spec.managed.roles`) for the warehouse read-only principal.

### Why

- GitOps-native lifecycle for the DB role (`present`/`absent` semantics).
- Operator reconciliation corrects drift from manual DB edits.
- Password can be reconciled from a Kubernetes Secret (`passwordSecret`).
- Status visibility through managed role reconciliation in Cluster status.

### Why not a Job (for now)

Kubernetes Job with SQL can be made idempotent, but introduces extra moving parts and weaker drift correction compared with operator-native role reconciliation.

## Important Caveat: Role vs Grants

`managed.roles` handles role lifecycle and attributes, but read-only behavior still depends on grants.

Conceptual split:

- Role existence and password: declarative via CNPG `managed.roles`
- Privilege model: explicit SQL grants/default privileges strategy

Suggested grant model to evaluate:

1. `CONNECT` on target database
2. `USAGE` on target schemas
3. `SELECT` on all existing tables
4. `ALTER DEFAULT PRIVILEGES` for future tables to keep read-only role current

## Generic Cloudflare Tunnel Direction

### Current state

Tunnel config is per app (duplicated pattern), not centralized.

### Suggested target architecture

Run one shared `cloudflared` deployment + one shared config source with ordered ingress routes.

High-level flow:

```text
Internet
   |
Cloudflare DNS (CNAMEs -> tunnel UUID)
   |
Cloudflare Tunnel (shared deployment)
   |
Ingress rules (first-match-wins)
   |------------------------------|
   v                              v
linkding service             streamlit service
```

### Routing guidance

- Prefer hostname-based routing per app (`linkding.jellylabs.xyz`, `sales.jellylabs.xyz`).
- Avoid path-based routing for Streamlit unless required (more app/path edge cases).
- Keep final fallback route: `http_status:404`.

### If replacing Linkding

Two valid options:

1. Keep tunnel infra, repoint existing Linkding hostname route to Streamlit service.
2. Keep Linkding route and add a new Streamlit hostname route.

Option 2 is less disruptive operationally.

## Open Questions

1. Should warehouse and warehouse-dev be upgraded from PG13 to PG15 now, or tracked as a follow-up change?
2. What exact schema scope should the read-only role cover (single schema vs all analytics schemas)?
3. Do we want one shared tunnel workload for all public/internal apps, or a smaller shared tunnel just for selected services first?

## Next Planning Step (if desired)

Create an OpenSpec change proposal that includes:

- CNPG `managed.roles` adoption for the warehouse read-only role
- declarative grant strategy for durable read-only access
- generic/shared Cloudflare tunnel configuration and route ownership model
- migration plan for any existing per-app tunnel manifests (including Linkding)
