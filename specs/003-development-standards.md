# 003 - Development Standards

Conventions and rules for writing Kubernetes manifests, managing secrets, databases, and the GitOps workflow in this cluster.

## Secret Management

- Never commit secrets to the repository
- Use External Secrets Operator with Azure Key Vault integration
- Reference secrets through ExternalSecret resources

## Database Standards

- All databases MUST use CloudNative-PG operator, never standalone PostgreSQL
- Database configurations located in `databases/data/<database-name>/`
- Each database has its own namespace following pattern `<app-name>-db`
- Database cluster naming: `<app-name>-db-production-cnpg-v1`
- Applications connect via service: `<cluster-name>-rw.<namespace>.svc.cluster.local:5432`
- Use PostgreSQL 15 image: `ghcr.io/cloudnative-pg/postgresql:15`

## GitOps Workflow

- All changes are deployed through Flux CD
- **IMPORTANT**: Flux reconciliation can only happen AFTER git changes are committed and pushed. Never attempt `flux reconcile` on uncommitted local changes.
- Use Renovate for automated dependency updates
- Follow Commitizen format for commit messages (conventional commits)
- Kustomize for environment-specific configurations

## Homepage Catalog Conventions

- Homepage-worthy applications MUST include an explicit Homepage presence decision in the same change.
- Ingress-backed Homepage apps MUST declare `gethomepage.dev/*` annotations on the ingress resource, whether the ingress is public or local-only.
- Homepage-worthy apps without an ingress MUST use a curated entry in `apps/base/homepage/configmap.yaml`.
- Helper workloads and backing components MAY be intentionally excluded from Homepage when they are not operator-facing.
- Local-only Homepage ingresses SHOULD use the hostname pattern `<app>.local`.
- Review new or changed application manifests for Homepage alignment alongside ingress, service, and label changes.
