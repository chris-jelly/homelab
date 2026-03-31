# Mealie Runbook

## External prerequisites

This repository does not manage Azure storage resources, Azure Key Vault secret values, or Cloudflare DNS records directly.

### Azure backup placement

- Resource group: `rg-jellyhomelab`
- Shared storage account: `sthomelabbackups`
- Backup container: `mealie`
- CNPG backup destination prefix: `https://sthomelabbackups.blob.core.windows.net/mealie/db/`
- `/app/data` backup destination prefix: `https://sthomelabbackups.blob.core.windows.net/mealie/files/`

Create or confirm this blob container and prefix layout:

- `mealie`
- `mealie/db/`
- `mealie/files/`

### Workload identity prerequisites

- Kubernetes `ServiceAccount` for file backups: `mealie-backup`
- Kubernetes `ServiceAccount` used by CNPG at runtime: `mealie-db-production-cnpg-v1`
- Azure managed identity: `id-mealie-backup`
- Azure federated identity credential for file backups: `fic-mealie-backup`
- Federated subject for file backups: `system:serviceaccount:mealie:mealie-backup`
- Additional Azure federated identity credential for CNPG: trust `system:serviceaccount:mealie:mealie-db-production-cnpg-v1`
- Service account annotation for the Azure UAMI client ID from `mealie_backup_uami_client_id`
- Required workload identity pod label so the webhook injects Azure federation settings for backup consumers
- `Storage Blob Data Contributor` role assignment scoped to the `mealie` container

If the cluster uses namespace-level conventions for workload identity, confirm the Mealie backup pods still inherit the right label and service account binding.

Use these OpenTofu outputs from the backup handoff stack when wiring the manifests:

- `mealie_backup_uami_client_id`
- `mealie_backup_uami_principal_id`
- `mealie_backup_container_name`
- `mealie_backup_db_prefix_url`
- `mealie_backup_files_prefix_url`

### Backup image prerequisite

The Mealie backup CronJob uses the published helper image from `homelab-images`:

```text
ghcr.io/chris-jelly/mealie-backup-azcopy:10.25.1-r1
```

If the image needs to change, update it in `homelab-images` and publish a new immutable tag before changing the reference in this repo.

The CronJob expects AzCopy `10.25.1` or later so `AZCOPY_AUTO_LOGIN_TYPE=WORKLOAD` works with Arc workload identity.

### Azure Key Vault prerequisites

Vault: `kv-jellyhomelabprod`

Create or confirm these secret names:

- `secret/mealie-openai-api-key` = Mealie OpenAI API key

### DNS prerequisite

- Create or confirm `mealie.jellylabs.xyz` in Cloudflare pointing at the shared tunnel target

This repo already expects Traefik to terminate app routing based on the ingress hostname.

## Rollout validation

Run these checks after Flux reconciles `apps/production/mealie` and `databases/data/mealie`.

1. Verify the deployment and database cluster become ready:

```bash
kubectl get deploy,svc,pvc,ingress,certificate -n mealie
kubectl get cluster,scheduledbackup,externalsecret -n mealie
```

2. Verify the Mealie app secret wiring:

```bash
kubectl get secret -n mealie mealie-db-app-credentials -o jsonpath='{.data.username}' | base64 -d && printf '\n'
kubectl get secret -n mealie mealie-secrets -o jsonpath='{.data.OPENAI_API_KEY}' | base64 -d | wc -c
```

3. Verify Homepage discovery metadata is present on the ingress:

```bash
kubectl get ingress -n mealie mealie -o jsonpath='{.metadata.annotations.gethomepage\.dev/name}{"\n"}'
kubectl get ingress -n mealie mealie -o jsonpath='{.metadata.annotations.gethomepage\.dev/pod-selector}{"\n"}'
```

4. Verify the app responds locally before testing through the public hostname:

```bash
kubectl port-forward -n mealie svc/mealie 9000:9000
curl -I http://127.0.0.1:9000/
```

5. Verify the public hostname after DNS propagation:

```bash
curl -I https://mealie.jellylabs.xyz
```

6. Verify workload identity wiring and scheduled backup resources exist:

```bash
kubectl get serviceaccount -n mealie mealie-backup -o yaml
kubectl get serviceaccount -n mealie mealie-db-production-cnpg-v1 -o yaml
kubectl get cronjob -n mealie mealie-backup -o jsonpath='{.spec.jobTemplate.spec.template.spec.serviceAccountName}{"\n"}'
kubectl get cronjob -n mealie mealie-backup -o jsonpath='{.spec.jobTemplate.spec.template.metadata.labels.azure\.workload\.identity/use}{"\n"}'
kubectl get pod -n mealie -l cnpg.io/cluster=mealie-db-production-cnpg-v1 -o jsonpath='{.items[0].spec.serviceAccountName}{"\n"}'
kubectl get cluster -n mealie mealie-db-production-cnpg-v1 -o jsonpath='{.spec.inheritedMetadata.labels.azure\.workload\.identity/use}{"\n"}'
kubectl get scheduledbackup -n mealie mealie-db-scheduled-backup -o yaml
```

Expected values:

- CronJob service account: `mealie-backup`
- CNPG pod service account: `mealie-db-production-cnpg-v1`

Azure must trust both subjects before backups can succeed.

7. Trigger and inspect the file backup job manually if needed:

```bash
kubectl create job --from=cronjob/mealie-backup -n mealie mealie-backup-manual-$(date +%s)
kubectl get jobs -n mealie
kubectl logs -n mealie job/$(kubectl get jobs -n mealie -o name | grep mealie-backup-manual | tail -n 1 | cut -d/ -f2)
```

8. Verify the backup consumers received workload identity settings and can see the Azure destination:

```bash
kubectl get pod -n mealie -l app.kubernetes.io/component=backup -o jsonpath='{.items[0].spec.serviceAccountName}{"\n"}'
kubectl get pod -n mealie -l app.kubernetes.io/component=backup -o jsonpath='{.items[0].metadata.labels.azure\.workload\.identity/use}{"\n"}'
kubectl exec -n mealie $(kubectl get pod -n mealie -l cnpg.io/cluster=mealie-db-production-cnpg-v1 -o jsonpath='{.items[0].metadata.name}') -- env | grep '^AZURE_'
kubectl exec -n mealie job/$(kubectl get jobs -n mealie -o name | grep mealie-backup-manual | tail -n 1 | cut -d/ -f2) -- env | grep '^AZURE_'
```

The CNPG pod should still show service account `mealie-db-production-cnpg-v1` even though the file-backup job uses `mealie-backup`.

9. Confirm the file-backup job can reach Blob Storage without a Kubernetes secret:

```bash
kubectl get externalsecret -n mealie
kubectl logs -n mealie job/$(kubectl get jobs -n mealie -o name | grep mealie-backup-manual | tail -n 1 | cut -d/ -f2) | grep -E 'Upload successful|Deleting old backup|Current backups'
```

The `mealie` namespace should no longer contain `mealie-backup-secrets` or `cnpg-azure-backup-creds`.

10. Confirm CNPG workload identity is healthy after the Azure-side federated credential update:

```bash
kubectl get cluster -n mealie mealie-db-production-cnpg-v1 -o jsonpath='{.status.conditions[?(@.type=="ContinuousArchiving")].status}{"\n"}'
kubectl get cluster -n mealie mealie-db-production-cnpg-v1 -o jsonpath='{.status.conditions[?(@.type=="ContinuousArchiving")].message}{"\n"}'
kubectl logs -n mealie $(kubectl get pod -n mealie -l cnpg.io/cluster=mealie-db-production-cnpg-v1 -o jsonpath='{.items[0].metadata.name}') --tail=100 | grep -E 'AADSTS700213|barman-cloud-check-wal-archive'
```

Success means the `ContinuousArchiving` condition becomes healthy and the CNPG logs stop reporting the federated subject mismatch.

## Recovery drill notes

These drills require a deployed, healthy Mealie environment and should be run on-cluster after backups are confirmed.

### CNPG recovery drill

1. Confirm at least one successful `Backup` object exists for `mealie-db-production-cnpg-v1`.
2. Create a temporary recovery cluster in the `mealie` namespace using the same backup object store and a distinct cluster name.
3. Verify the recovered cluster reaches primary state.
4. Connect to the recovered database and confirm expected Mealie tables and recent data exist.
5. Remove the temporary recovery cluster after validation.

### `/app/data` restore drill

1. Download a recent `mealie-app-data-backup-*.tar.gz` archive from the `mealie/files/` prefix.
2. Start a temporary pod with the `mealie-data` PVC mounted read-write.
3. Extract the archive into a scratch directory first and inspect contents.
4. Restore into the mounted PVC only after confirming ownership and expected files.
5. Restart the Mealie deployment and confirm uploads and generated assets still load correctly.

## Rollback notes

If workload identity rollout fails:

1. Revert `apps/production/mealie/backup-cronjob.yaml`, `apps/production/mealie/backup-config-configmap.yaml`, and `apps/production/mealie/backup-script-configmap.yaml` to the prior secret-based backup flow.
2. Revert `databases/data/mealie/database.yaml` to the previous CNPG backup credential configuration.
3. Restore `apps/production/mealie/backup-externalsecret.yaml` and `databases/data/mealie/backup-externalsecret.yaml` if the rollback returns to secret-based Blob access.
4. Reconcile Flux and confirm the CronJob and CNPG cluster return to a healthy state before removing any Blob artifacts.
5. If the corrected two-subject model is the problem, remove the explicit `mealie-backup` service account manifest and revert the CronJob to the prior secret-based path before touching the CNPG cluster.
6. If the Azure side is the problem, revert or remove the additional federated credential for `system:serviceaccount:mealie:mealie-db-production-cnpg-v1` and confirm CNPG returns to the previous auth model.
7. Keep the Azure `id-mealie-backup`, the CronJob federated credential, the CNPG federated credential, and the `mealie` container in place unless you are intentionally dismantling the backup integration.
