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
