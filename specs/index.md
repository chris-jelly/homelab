# Specs

Runbooks and specifications for the homelab cluster.

| Spec | Description |
|------|-------------|
| [001 - Cluster Bootstrap](./001-cluster-bootstrap.md) | Runbook for bootstrapping the k3s cluster from a fresh Ubuntu 24.04 install, including Flux CD, SOPS, and Azure Key Vault credentials |
| [002 - Data Resilience](./002-data-resilience.md) | Plan for backup and recovery of all stateful workloads using CNPG continuous backups and CronJob-based file archival to Azure Blob Storage |
