# Shared Cloudflare Edge Migration Notes

## DNS cutover (operational)

Update Cloudflare DNS records to point both hostnames at the shared tunnel target.

- `linkding.jellylabs.xyz` -> shared tunnel target
- `streamlit.jellylabs.xyz` -> shared tunnel target

This repository does not manage Cloudflare DNS records yet. Apply DNS changes manually in Cloudflare.

## Cloudflare tunnel origin setting

In Cloudflare Tunnel Public Hostnames, set the origin service URL to the Traefik service:

- `http://traefik.kube-system.svc.cluster.local:80`

Do not point tunnel hostnames directly at app-local short names such as `http://linkding:9090` or `http://sales-pipeline-pulse:8501` from the shared tunnel namespace. Use Traefik as the single origin and let Kubernetes `Ingress` rules handle hostname routing.

## Validation checklist

After Flux reconciliation and DNS propagation:

1. `curl -I https://linkding.jellylabs.xyz`
2. `curl -I https://streamlit.jellylabs.xyz`
3. Confirm both return expected HTTP status and app responses.
4. Confirm `kubectl get deploy -n linkding cloudflared` returns not found.

## Rollback

If cutover validation fails:

1. Restore `apps/production/linkding/cloudflare.yaml` in `apps/production/linkding/kustomization.yaml`.
2. Re-add `linkding-cloudflare-tunnel-credentials` mapping in `apps/production/linkding/external-secrets.yaml`.
3. Revert DNS for `linkding.jellylabs.xyz` to the previous app-local tunnel target if needed.
4. Reconcile Flux and re-run `curl` checks.
