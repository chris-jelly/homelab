# Streamlit Upstream Handoff Packet

Date: 2026-03-04

## Runtime Evidence

- Namespace/workload: `streamlit` / `deployment/sales-pipeline-pulse`
- Image: `ghcr.io/chris-jelly/de-streamlit-sales-analytics/streamlit-sales-pipeline-pulse:main`
- Image digest observed: `sha256:c0d50b9d74f918ef09cae1dbbccd80ebdb03965e6cd836c7cb01b211e6dab6d8`
- Effective startup command:

```text
/app/.venv/bin/python /app/.venv/bin/streamlit run streamlit_app/app.py --server.port=8501 --server.address=0.0.0.0
```

- Working directory / import context observed:

```text
cwd=/app
sys.path[0:5]=['', '/usr/local/lib/python313.zip', '/usr/local/lib/python3.13', '/usr/local/lib/python3.13/lib-dynload', '/app/.venv/lib/python3.13/site-packages']
```

- Current rendered secret value starts with:

```text
postgresql://
```

## Observed Errors

1. Backend configuration error in app UI:
   - `SALES_WAREHOUSE_URL must use postgresql+psycopg:// (psycopg v3)`
2. Prior runtime traceback (captured in cluster logs):

```text
ModuleNotFoundError: No module named 'streamlit_app'
File "/app/streamlit_app/app.py", line 8
from streamlit_app.config import Settings, from_env
```

## Suggested Upstream Follow-Up

- Validate app startup/import behavior with container runtime context equivalent to homelab deployment.
- Confirm runtime expectations for package import resolution when project is not installed as a site package.
- Re-test with DSN `postgresql+psycopg://` once homelab rollout applies manifest update.
