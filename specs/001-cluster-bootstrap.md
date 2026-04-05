# 001 - Cluster Bootstrap Runbook

How to go from a fresh Ubuntu 24.04 Server install on the control plane node to a fully operational k3s cluster with Flux CD managing all workloads.

## Prerequisites

- Raspberry Pi 5 (control plane) with Ubuntu 24.04 Server installed
- SSH access to the Pi
- Worker nodes networked and reachable from the control plane
- Azure CLI 2.64 or later with the `connectedk8s` extension 1.10.0 or later
- Access to subscription `10a32563-6c11-4d82-a986-be076562eb33`
- Azure Service Principal credentials (ClientID + ClientSecret) for External Secrets Operator
- This repo cloned on your local machine

The homelab Azure footprint already exists in `rg-jellyhomelab` in `Canada Central`. Reuse the existing resource group, Key Vaults, storage account, and Arc-connected cluster resource instead of creating parallel resources during bootstrap.

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

The Pi is control-plane only. Prevent workloads from being scheduled on it:

```bash
sudo k3s kubectl taint nodes $(hostname) node-role.kubernetes.io/control-plane:NoSchedule
```

## 2. Join Worker Nodes

On the control plane, get the node token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### If workers had a previous k3s agent installed

The old agent cannot authenticate against the new control plane because the cluster CA changed. Uninstall first, then re-join:

```bash
/usr/local/bin/k3s-agent-uninstall.sh
```

This removes the k3s agent binary and config but does **not** wipe local storage such as `/var/lib/rancher/k3s/storage/`. Old PV data remains on disk as orphaned directories. Kubernetes will not reuse it. Clean it up manually later if you need the space.

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

## 4. Connect the Cluster to Azure Arc and Enable Workload Identity

These steps establish the Arc-managed OIDC issuer that Azure-integrated workloads use for Microsoft Entra workload identity federation.

Set the shared values first:

```bash
export ARC_RESOURCE_GROUP="rg-jellyhomelab"
export ARC_LOCATION="canadacentral"
export ARC_CLUSTER_NAME="HomelabArc"
```

### Register the required Azure resource providers

Provider registration is subscription-scoped. If these providers are already registered, this is a no-op.

```bash
for ns in Microsoft.Kubernetes Microsoft.KubernetesConfiguration Microsoft.ExtendedLocation; do
  az provider register --namespace "${ns}" --wait
done
```

### Connect the cluster if needed

If the Arc-connected cluster resource already exists, skip the `connect` call and move to the `show` and `update` steps.

```bash
az connectedk8s show \
  --name "${ARC_CLUSTER_NAME}" \
  --resource-group "${ARC_RESOURCE_GROUP}"
```

Create it only when that `show` command fails:

```bash
az connectedk8s connect \
  --name "${ARC_CLUSTER_NAME}" \
  --resource-group "${ARC_RESOURCE_GROUP}" \
  --location "${ARC_LOCATION}" \
  --distribution k3s \
  --enable-oidc-issuer \
  --enable-workload-identity
```

### Ensure OIDC issuer and workload identity are enabled

Run the update command even if the cluster was connected earlier. It is the safest way to converge the Arc feature flags.

```bash
az connectedk8s update \
  --name "${ARC_CLUSTER_NAME}" \
  --resource-group "${ARC_RESOURCE_GROUP}" \
  --enable-oidc-issuer \
  --enable-workload-identity
```

### Verify the Arc-connected cluster resource and agents

```bash
az connectedk8s show \
  --name "${ARC_CLUSTER_NAME}" \
  --resource-group "${ARC_RESOURCE_GROUP}" \
  --query '{name:name,location:location,provisioningState:provisioningState,oidcIssuer:oidcIssuerProfile.issuerUrl,workloadIdentity:securityProfile.workloadIdentity.enabled}' \
  -o yaml

kubectl get pods -n azure-arc
```

### Retrieve the Arc-managed issuer URL

```bash
export ARC_OIDC_ISSUER="$(az connectedk8s show \
  --name "${ARC_CLUSTER_NAME}" \
  --resource-group "${ARC_RESOURCE_GROUP}" \
  --query 'oidcIssuerProfile.issuerUrl' \
  -o tsv)"

printf '%s\n' "${ARC_OIDC_ISSUER}"
```

### Configure k3s to mint tokens with the Arc issuer

If `/etc/rancher/k3s/config.yaml` does not exist yet, create it. If it already exists, merge these settings into the existing file instead of overwriting other config.

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<EOF
kube-apiserver-arg:
  - "service-account-issuer=${ARC_OIDC_ISSUER}"
  - "service-account-max-token-expiration=24h"
EOF
```

Restart k3s so new projected service account tokens use the Arc issuer:

```bash
sudo systemctl restart k3s
sudo systemctl status k3s --no-pager
```

### Validate the Arc issuer, JWKS, and fresh token claims

Validate the discovery document:

```bash
python - <<'PY'
import json
import os
import urllib.request

url = os.environ['ARC_OIDC_ISSUER'] + '.well-known/openid-configuration'
with urllib.request.urlopen(url) as response:
    data = json.load(response)
print(json.dumps({'issuer': data['issuer'], 'jwks_uri': data['jwks_uri']}, indent=2))
PY
```

Validate the JWKS endpoint:

```bash
python - <<'PY'
import json
import os
import urllib.request

discovery_url = os.environ['ARC_OIDC_ISSUER'] + '.well-known/openid-configuration'
with urllib.request.urlopen(discovery_url) as response:
    jwks_uri = json.load(response)['jwks_uri']
with urllib.request.urlopen(jwks_uri) as response:
    jwks = json.load(response)
print(json.dumps({'key_count': len(jwks.get('keys', [])), 'kids': [key.get('kid') for key in jwks.get('keys', [])]}, indent=2))
PY
```

Mint a fresh service account token and confirm the `iss` claim matches the Arc issuer exactly:

```bash
export TOKEN="$(kubectl create token default -n default --duration=10m)"

python - <<'PY'
import base64
import json
import os

payload = os.environ['TOKEN'].split('.')[1]
payload += '=' * (-len(payload) % 4)
claims = json.loads(base64.urlsafe_b64decode(payload))
print(json.dumps({'iss': claims.get('iss'), 'sub': claims.get('sub'), 'aud': claims.get('aud')}, indent=2))
PY
```

### Rollback and cleanup

If k3s fails after the issuer change, restore the previous `/etc/rancher/k3s/config.yaml` content and restart k3s:

```bash
sudo systemctl restart k3s
```

If the Arc integration itself must be removed, delete the connected-cluster resource with Azure CLI so Azure-side resources and in-cluster agents are cleaned up together:

```bash
az connectedk8s delete \
  --name "${ARC_CLUSTER_NAME}" \
  --resource-group "${ARC_RESOURCE_GROUP}" \
  --yes
```

### Standard Azure workload identity pattern

Use this pattern for future Azure-integrated workloads after Arc is enabled:

1. Set `kubernetes_oidc_issuer_url` in `~/git/jellylabs-dsc/infrastructure/azure/homelab/` to `ARC_OIDC_ISSUER` before `tofu apply`.
2. Create one Azure user-assigned managed identity and one federated credential per workload boundary.
3. Use subject format `system:serviceaccount:<namespace>:<service-account>` in the federated credential.
4. Annotate the Kubernetes service account with `azure.workload.identity/client-id: <managed-identity-client-id>`.
5. Add pod label `azure.workload.identity/use: "true"` to each workload that should be mutated by the Arc workload identity webhook.

The current Mealie backup handoff in `~/git/jellylabs-dsc/infrastructure/azure/homelab/MEALIE_BACKUP_HANDOFF.md` follows this pattern.

## 5. Bootstrap Flux CD

```bash
export GITHUB_TOKEN=<your-github-pat>

flux bootstrap github \
  --owner=chris-jelly \
  --repository=homelab \
  --branch=main \
  --path=clusters/production \
  --personal
```

This installs the Flux controllers, creates the GitRepository source and root Kustomization, and points Flux at `clusters/production/`. Run this from any machine with `kubectl` access to the cluster. It does not need to run on the control plane node.

The GitHub token requires repo write access so `flux bootstrap` can push updates to the Flux manifests in the repo. Although the repo is public, bootstrap needs to commit changes such as Flux version bumps or URL updates. After bootstrap, Flux reads from the public repo without any token.

`flux bootstrap` is idempotent and safe to run against a repo that was previously bootstrapped. It reconciles against the existing manifests in `clusters/production/flux-system/` and commits updates if anything has changed.

## 6. Prepare External Secrets Operator Key Vault access

External Secrets Operator now authenticates to `kv-jellyhomelabprod` with `authType: WorkloadIdentity` through a referenced service account instead of the old `azure-creds` bootstrap secret.

Before expecting Key Vault-backed `ExternalSecret` reconciliation to succeed, make sure the Azure workload identity prerequisites are already in place:

1. Arc workload identity is enabled and the cluster issuer URL is stable.
2. `~/git/jellylabs-dsc/infrastructure/azure/homelab/` has been applied with the current `kubernetes_oidc_issuer_url`.
3. The Azure stack created the federated credential for subject `system:serviceaccount:external-secrets:azure-kv-store-reader` and granted that identity `Key Vault Secrets User` on `kv-jellyhomelabprod`.

The GitOps-managed `ClusterSecretStore` uses this referenced service account contract:

- Store: `ClusterSecretStore/azure-kv-store`
- Vault: `https://kv-jellyhomelabprod.vault.azure.net/`
- Auth type: `WorkloadIdentity`
- Referenced service account: `ServiceAccount/azure-kv-store-reader` in namespace `external-secrets`

There is no longer a supported bootstrap step that creates `Secret/azure-creds` in the cluster for External Secrets Operator.

## 7. Verify Flux Reconciliation

Once Flux is bootstrapped and the Azure workload identity prerequisites are ready, reconciliation should proceed. Monitor progress:

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

```text
flux-system
  |-- infra-controllers    (cert-manager, cloudnative-pg, external-secrets, renovate)
  |     '-- infra-configs  (ClusterIssuers, ClusterSecretStores)
  |-- monitoring-controllers (kube-prometheus-stack)
  |     '-- monitoring-configs (Grafana dashboards)
  |-- databases            (CloudNative-PG clusters: airflow, warehouse)
        '-- apps           (audiobookshelf, linkding, actual-budget, airflow, homepage)
```

`infra-controllers`, `monitoring-controllers`, and `databases` have no inter-dependencies and reconcile in parallel. Their downstream dependents wait until the parent is ready.

## 8. Post-Bootstrap Verification

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

Check the referenced service account annotation:

```bash
kubectl get serviceaccount azure-kv-store-reader -n external-secrets -o yaml
```

Check the `ClusterSecretStore` status:

```bash
kubectl get clustersecretstore azure-kv-store -o yaml
```

If the store is not ready, confirm the referenced service account subject matches Azure exactly:

```text
system:serviceaccount:external-secrets:azure-kv-store-reader
```

Also confirm the backing Azure identity still has `Key Vault Secrets User` on `kv-jellyhomelabprod`.

### Arc workload identity validation

Confirm the Arc cluster resource is healthy:

```bash
az connectedk8s show \
  --name "HomelabArc" \
  --resource-group "rg-jellyhomelab" \
  --query '{connectivityStatus:connectivityStatus,provisioningState:provisioningState,issuer:oidcIssuerProfile.issuerUrl,workloadIdentity:securityProfile.workloadIdentity.enabled}' \
  -o yaml
```

Confirm the Arc agents are running:

```bash
kubectl get pods -n azure-arc
```

If tokens still show the old issuer, inspect the k3s config and mint a fresh token after the restart:

```bash
sudo cat /etc/rancher/k3s/config.yaml
kubectl create token default -n default --duration=10m
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
| Arc resource group | `rg-jellyhomelab` |
| Arc location | `canadacentral` |
| Arc connected cluster | `HomelabArc` |
| Azure KV (primary) | `kv-jellyhomelabprod.vault.azure.net` |
| ESO Key Vault auth | `WorkloadIdentity` via `ServiceAccount/azure-kv-store-reader` |
