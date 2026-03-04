# Sales Pipeline Pulse Runbook

## Rollout Validation

Use these checks after Flux applies changes for `apps/production/sales-pipeline-pulse`.

1. Confirm the effective warehouse DSN scheme is psycopg v3:

```bash
kubectl get secret -n streamlit sales-pipeline-pulse-secrets -o jsonpath='{.data.SALES_WAREHOUSE_URL}' | base64 -d
```

Expected prefix: `postgresql+psycopg://`

2. Confirm the app serves a request path and not only probe endpoints:

```bash
kubectl port-forward -n streamlit svc/sales-pipeline-pulse 8501:8501
curl -i http://127.0.0.1:8501/
```

3. Review recent runtime logs for uncaught app execution errors:

```bash
kubectl logs -n streamlit deploy/sales-pipeline-pulse --tail=200
```

Do not treat readiness/liveness probe success as sufficient rollout validation.

## Upstream Handoff Checklist (Streamlit Repo)

When runtime failures persist after homelab wiring fixes, provide this packet to the Streamlit repository owner:

- Full traceback from `kubectl logs`.
- Deployed image reference (tag and digest).
- Effective startup command from `/proc/1/cmdline`.
- Effective working directory and import context (`pwd`, top `sys.path` entries).
- Current `SALES_WAREHOUSE_URL` prefix evidence.

Command snippets:

```bash
kubectl describe pod -n streamlit -l app.kubernetes.io/name=sales-pipeline-pulse
kubectl exec -n streamlit deploy/sales-pipeline-pulse -- sh -lc 'xargs -0 echo < /proc/1/cmdline'
kubectl exec -n streamlit deploy/sales-pipeline-pulse -- python -c "import os,sys; print(os.getcwd()); print(sys.path[:5])"
```
