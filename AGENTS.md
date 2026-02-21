# AGENTS.md

## First Step

Before starting any task:

1. Load the `kubernetes-specialist` skill for general Kubernetes best practices (manifest structure, security hardening, workload patterns, networking, Helm, GitOps, etc.)
2. Read `specs/index.md` to review the runbooks and specifications specific to this cluster. Specs contain repo-specific conventions and operational decisions that may override or extend the general practices from the skill.

This ensures you have both general Kubernetes expertise and knowledge of this cluster's specific decisions before making changes.

## Project Overview

This is a Kubernetes homelab managed with GitOps principles using Flux CD. The infrastructure is a Raspberry Pi for the control plane and various laptops for worker nodes, each ~16GB of RAM. All applications and infrastructure components are defined as Kubernetes manifests and deployed through Flux CD.

## Repository Structure

- **apps/**: Application deployments organized by environment (base/production) with Kustomize overlays
- **infrastructure/**: Core infrastructure components (cert-manager, external-secrets, cloudnative-pg, renovate)
- **monitoring/**: Observability stack (kube-prometheus-stack with Grafana/Prometheus)
- **databases/**: Database configurations using CloudNative-PG operator
- **clusters/**: Cluster-specific Flux configurations that reference the above layers
- **specs/**: Runbooks and specifications for cluster conventions and operations
