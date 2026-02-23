## 1. Upgrade CNPG Operator

- [x] 1.1 Bump chart version in `infrastructure/controllers/base/cloudnative-pg/release.yaml` from `0.24.0` to `0.27.1`
- [x] 1.2 Commit and push the CNPG operator version bump
- [x] 1.3 Wait for Flux to reconcile the `infra-controllers` kustomization and verify the CNPG operator pod is running the new version (`kubectl get pods -n cloudnative-pg`)
- [x] 1.4 Verify warehouse clusters are healthy after operator upgrade (`kubectl get cluster -A` — both `warehouse-db-production-cnpg-v1` and `warehouse-db-dev-cnpg-v1` should show healthy status)

## 2. Tear Down Existing Airflow Deployment

- [x] 2.1 Suspend the Airflow HelmRelease to prevent Flux reconciliation: `flux suspend hr airflow -n airflow`
- [x] 2.2 Delete the Airflow HelmRelease to uninstall the Helm release: `flux delete hr airflow -n airflow`
- [x] 2.3 Delete the Airflow CNPG Cluster to drop the database and PVC: `kubectl delete cluster airflow-db-production-cnpg-v1 -n airflow`
- [x] 2.4 Verify teardown is complete — no Airflow pods running, no CNPG cluster in airflow namespace (`kubectl get pods -n airflow`, `kubectl get cluster -n airflow`)

## 3. Rewrite Airflow HelmRelease Values

- [x] 3.1 Update chart version from `1.15.0` to `1.19.0` in `apps/base/airflow/release.yaml`
- [x] 3.2 Replace `webserver` section with `apiServer` section (resources: 200m/1000m CPU, 1Gi/2Gi memory)
- [x] 3.3 Move `webserver.defaultUser.enabled: true` into `createUserJob.defaultUser.enabled: true`
- [x] 3.4 Replace `ingress.web` with `ingress.apiServer` (same host, TLS, ingressClassName settings)
- [x] 3.5 Remove deprecated config vars: `AIRFLOW__API__AUTH_BACKENDS` and `AIRFLOW__WEBSERVER__EXPOSE_CONFIG`
- [x] 3.6 Remove legacy git-sync overrides: `containerName` and `uid` from `dags.gitSync`
- [x] 3.7 Verify remaining values are correct: executor, scheduler resources, worker resources, postgresql disabled, data.metadataSecretName, pgbouncer/redis/flower disabled, Flux job hooks, extraEnvFrom

## 4. Commit and Deploy

- [x] 4.1 Commit the updated Airflow manifests
- [x] 4.2 Push and let Flux reconcile — it will recreate the CNPG cluster and deploy the new HelmRelease
- [x] 4.3 Monitor the CNPG cluster creation: `kubectl get cluster -n airflow` — wait for it to become healthy
- [x] 4.4 Monitor the HelmRelease reconciliation: `flux get hr airflow -n airflow` — wait for it to show `Ready`
- [x] 4.5 Verify the `migrateDatabaseJob` ran successfully against the fresh database
- [x] 4.6 Verify the `createUserJob` ran and admin user was created

## 5. Post-Deploy Validation

- [x] 5.1 Verify Airflow API server pod is running: `kubectl get pods -n airflow`
- [x] 5.2 Verify scheduler pod is running and healthy
- [x] 5.3 Verify git-sync sidecar is pulling DAGs (check git-sync container logs)
- [x] 5.4 Access Airflow UI at `https://airflow.jellylabs.xyz` and confirm login works with admin credentials
- [x] 5.5 Confirm Airflow version shows 3.x in the UI footer/about page
- [x] 5.6 Check scheduler logs for DAG import errors (expected — DAGs may need AF3 updates, but the deployment itself should be healthy)
- [x] 5.7 Verify warehouse CNPG clusters are still healthy after all changes
