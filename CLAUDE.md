# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes homelab managed with GitOps principles using Flux CD. The infrastructure is a raspberry Pi for the control plane and various Laptops for worker nodes, each ~16GB of RAM.
All applications and infrastructure components are defined as Kubernetes manifests and deployed through Flux CD.

## Architecture

The repository follows a layered GitOps structure:

- **apps/**: Application deployments organized by environment (base/production) with Kustomize overlays
- **infrastructure/**: Core infrastructure components (cert-manager, external-secrets, cloudnative-pg, renovate)
- **monitoring/**: Observability stack (kube-prometheus-stack with Grafana/Prometheus)
- **databases/**: Database configurations using CloudNative-PG operator
- **clusters/**: Cluster-specific Flux configurations that reference the above layers

## Common Commands

### Environment Setup
```bash
# Install all required tools using mise
mise install

# Trust the mise configuration
mise trust mise.toml
```

### Kubernetes Operations
```bash
# View cluster resources with k9s
k9s

# Apply Kubernetes manifests
kubectl apply -f <manifest-file>

# Check Flux reconciliation status
flux get all

# Force reconciliation of a specific resource
flux reconcile <resource-type> <resource-name>
```

### Available Tools (via mise.toml)
- `kubectl`: Kubernetes CLI
- `flux2`: Flux CLI for GitOps operations
- `helm`: Helm package manager
- `k9s`: Terminal-based Kubernetes UI
- `cloudflared`: Cloudflare tunnel client

The KUBECONFIG is set to `./config.yaml` in the project root.

## Development Standards

### Kubernetes Manifests
- Use YAML format with 2-space indentation
- Include standard Kubernetes labels:
  - `app.kubernetes.io/name`
  - `app.kubernetes.io/instance` 
  - `app.kubernetes.io/part-of`
- Define resource requests and limits explicitly
- Organize applications by namespaces

### Secret Management
- Never commit secrets to the repository
- Use External Secrets Operator with Azure Key Vault integration
- Reference secrets through ExternalSecret resources

### Database Standards
- All databases MUST use CloudNative-PG operator, never standalone PostgreSQL
- Database configurations located in `databases/data/<database-name>/`
- Each database has its own namespace following pattern `<app-name>-db`
- Database cluster naming: `<app-name>-db-production-cnpg-v1`
- Applications connect via service: `<cluster-name>-rw.<namespace>.svc.cluster.local:5432`
- Use PostgreSQL 15 image: `ghcr.io/cloudnative-pg/postgresql:15`

### GitOps Workflow
- All changes deployed through Flux CD
- **IMPORTANT**: Flux reconciliation can only happen AFTER git changes are committed and pushed
- Never attempt `flux reconcile` on uncommitted local changes
- Use Renovate for automated dependency updates
- Follow Commitizen format for commit messages (conventional commits)
- Kustomize for environment-specific configurations

## Key Infrastructure Components

- **Flux CD**: GitOps controller managing deployments
- **Cert-Manager**: Automatic TLS certificate management with Cloudflare DNS
- **External Secrets Operator**: Secret synchronization from Azure Key Vault
- **CloudNative-PG**: PostgreSQL operator for database management
- **Kube-Prometheus-Stack**: Monitoring with Prometheus, Grafana, and Alertmanager
- **Renovate**: Automated dependency updates via CronJob

## Current Applications

- **Audiobookshelf**: Self-hosted audiobook and podcast server
- **Linkding**: Minimal bookmark manager
- **Airflow**: Workflow orchestration platform with git-sync for DAG deployment
