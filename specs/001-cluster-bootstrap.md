# 001 - Cluster Bootstrap Runbook

How to go from a fresh Ubuntu 24.04 Server install on the control plane node to a fully operational k3s cluster with Flux CD managing all workloads.

## Prerequisites

- Raspberry Pi 5 (control plane) with Ubuntu 24.04 Server installed
- SSH access to the Pi
- Worker nodes networked and reachable from the control plane
- Azure Service Principal credentials (ClientID + ClientSecret) for External Secrets Operator
- This repo cloned on your local machine

## 1. Install k3s on the Control Plane

SSH into the Pi and install k3s as a server node:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644
```

Wait for k3s to be ready:

```bash
sudo k3s kubectl get nodes
```

The control plane should show as `Ready` within a minute or two.

### Taint control plane to prevent workload scheduling

The Pi is control-plane only — prevent workloads from being scheduled on it:

```bash
sudo k3s kubectl taint nodes $(hostname) node-role.kubernetes.io/control-plane:NoSchedule
```

## 2. Join Worker Nodes

On the control plane, get the node token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### If workers had a previous k3s agent installed

The old agent can't authenticate against the new control plane (different cluster CA). Uninstall first, then re-join:

```bash
/usr/local/bin/k3s-agent-uninstall.sh
```

This removes the k3s agent binary and config but does **not** wipe local storage (e.g., `/var/lib/rancher/k3s/storage/`). Old PV data will remain on disk as orphaned directories — Kubernetes will not reuse them. If you need to reclaim disk space, clean them up manually after the new cluster is running.

### Join workers to the new control plane

On each worker node:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<CONTROL_PLANE_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> sh -s - agent
```

Verify all nodes are joined:

```bash
sudo k3s kubectl get nodes
```

## 3. Set Up Local kubectl Access

Copy the kubeconfig from the control plane to your local machine:

```bash
scp <user>@<CONTROL_PLANE_IP>:/etc/rancher/k3s/k3s.yaml ./config.yaml
```

Edit `config.yaml` and replace `127.0.0.1` with the actual control plane IP:

```bash
sed -i 's/127.0.0.1/<CONTROL_PLANE_IP>/g' config.yaml
```

The repo's `mise.toml` sets `KUBECONFIG=./config.yaml`, so from the repo root:

```bash
mise trust mise.toml
mise install
kubectl get nodes
```

## 4. Bootstrap Flux CD

```bash
export GITHUB_TOKEN=<your-github-pat>

flux bootstrap github \
  --owner=chris-jelly \
  --repository=homelab \
  --branch=main \
  --path=clusters/production \
  --personal
```

This installs the Flux controllers, creates the GitRepository source and root Kustomization, and points Flux at `clusters/production/`. Run this from any machine with `kubectl` access to the cluster — it does not need to run on the control plane node.

The GitHub token requires repo write access so `flux bootstrap` can push updates to the Flux manifests in the repo. Although the repo is public (no credentials needed for ongoing reads), bootstrap needs to commit changes like Flux version bumps or URL updates. After bootstrap, Flux reads from the public repo without any token.

`flux bootstrap` is idempotent and safe to run against a repo that was previously bootstrapped. It will reconcile against the existing manifests in `clusters/production/flux-system/` and commit updates if anything has changed.

## 5. Create the Azure Key Vault Credentials Secret

External Secrets Operator authenticates to Azure Key Vault using a ServicePrincipal. This secret must exist **before** `infra-configs` reconciles, since the `ClusterSecretStore` resources reference it.

```bash
kubectl create namespace external-secrets

kubectl create secret generic azure-creds \
  --namespace=external-secrets \
  --from-literal=ClientID=<AZURE_SP_CLIENT_ID> \
  --from-literal=ClientSecret=<AZURE_SP_CLIENT_SECRET>
```

This secret is referenced by both `ClusterSecretStore` resources:
- `azure-kv-store` (vault: `kv-jellyhomelabprod.vault.azure.net`)
- `azure-kv-work-integrations-store` (vault: `kv-work-integrations.vault.azure.net`)

Both use tenant ID `3c78a8ad-6f4f-45a0-bec9-8538f870a693`.

> **Important:** This is a manual bootstrap secret that cannot be managed by External Secrets itself (chicken-and-egg). Keep the credentials stored securely outside the cluster (e.g., in a password manager).

## 6. Verify Flux Reconciliation

Once the secrets are in place, Flux will begin reconciling. Monitor progress:

```bash
# Overall status
flux get all

# Watch kustomizations reconcile
flux get kustomizations --watch

# Check specific components
flux get kustomizations infra-controllers
flux get kustomizations infra-configs
flux get kustomizations monitoring-controllers
flux get kustomizations monitoring-configs
flux get kustomizations databases
flux get kustomizations apps
```

### Expected reconciliation order

The dependency chain defined in `clusters/production/` is:

```
flux-system
  |-- infra-controllers    (cert-manager, cloudnative-pg, external-secrets, renovate)
  |     '-- infra-configs  (ClusterIssuers, ClusterSecretStores)
  |-- monitoring-controllers (kube-prometheus-stack)
  |     '-- monitoring-configs (Grafana dashboards)
  |-- databases            (CloudNative-PG clusters: airflow, warehouse, warehouse-dev)
        '-- apps           (audiobookshelf, linkding, actual-budget, airflow, homepage)
```

`infra-controllers` and `monitoring-controllers` and `databases` have no inter-dependencies and will reconcile in parallel. Their downstream dependents wait until the parent is ready.

## 7. Post-Bootstrap Verification

Once all kustomizations show `Ready`:

```bash
# All pods running
kubectl get pods --all-namespaces

# Certificates issued
kubectl get certificates --all-namespaces

# External secrets synced
kubectl get externalsecrets --all-namespaces

# Database clusters healthy
kubectl get clusters.postgresql.cnpg.io --all-namespaces

# Helm releases reconciled
kubectl get helmreleases --all-namespaces
```

## Troubleshooting

### Flux kustomization stuck

```bash
# Force reconciliation
flux reconcile kustomization <name> --with-source

# Check events
kubectl events -n flux-system --for kustomization/<name>
```

### External secrets not syncing

Check that the `azure-creds` secret exists and has correct values:

```bash
kubectl get secret azure-creds -n external-secrets -o jsonpath='{.data}' | jq
```

Check the ClusterSecretStore status:

```bash
kubectl get clustersecretstore -o yaml
```

### k3s issues on the Pi

```bash
# Check k3s service status
sudo systemctl status k3s

# View k3s logs
sudo journalctl -u k3s -f

# Restart k3s
sudo systemctl restart k3s
```

## Quick Reference

| Item | Value |
|------|-------|
| Git repo | `https://github.com/chris-jelly/homelab` |
| Branch | `main` |
| Flux path | `clusters/production/` |
| Azure tenant ID | `3c78a8ad-6f4f-45a0-bec9-8538f870a693` |
| Azure KV (primary) | `kv-jellyhomelabprod.vault.azure.net` |
| Azure KV (work) | `kv-work-integrations.vault.azure.net` |
| Azure creds secret | `azure-creds` in `external-secrets` namespace |
