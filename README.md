# apache-superset

# Superset on Kubernetes — Deployment Runbook

This documents how Superset (Helm chart `superset-0.22.6`, Superset `6.1.0`) was
deployed into the `pms-superset` namespace, the issues hit along the way, and
the exact fixes. Keep this alongside `values.yaml` for future re-deploys.

---

## Prerequisites

- A working Kubernetes cluster with `kubectl` access
- Helm 3 installed
- A default `StorageClass` available (this deployment uses **Longhorn**)
- The Superset Helm chart cloned locally, chart root containing `Chart.yaml`

```bash
cd ~/superset/helm/superset
ls Chart.yaml values.yaml   # sanity check you're in the right directory
```

---

## Key values.yaml overrides

All overrides live directly in `values.yaml` (not a separate `values-custom.yaml`).
The important ones, and *why* each exists:

```yaml
# Fixes: psycopg2 driver missing from Superset's venv at runtime.
# The base image ships `uv` as its package manager, and the app venv
# (/app/.venv) has no pip binary of its own — plain `pip install` or
# `python -m pip install` both fail. `uv pip install --python` is what works.
bootstrapScript: |
  #!/bin/bash
  uv pip install --python /app/.venv/bin/python psycopg2-binary
  if [ ! -f ~/bootstrap ]; then echo "Running Superset with uid {{ .Values.runAsUser }}" > ~/bootstrap; fi

service:
  type: NodePort
  port: 8088
  nodePort:
    http: 30088   # NOTE: this chart expects a nested map, not a flat integer

ingress:
  enabled: false

init:
  enabled: true
  loadExamples: true
  createAdmin: true
  adminUser:
    username: admin
    firstname: Superset
    lastname: Admin
    email: admin@superset.com
    password: admin        # CHANGE after first login

configOverrides:
  secret: |
    SECRET_KEY = "REPLACE_WITH_YOUR_OWN_GENERATED_SECRET"   # openssl rand -base64 42

postgresql:
  enabled: true
  auth:
    username: superset
    password: "REPLACE_WITH_YOUR_OWN_PASSWORD"
    database: superset
  primary:
    persistence:
      enabled: true
      storageClass: longhorn
      size: 8Gi

redis:
  enabled: true
  architecture: standalone   # NOTE: chart default is `replication` if left commented out —
                              # will otherwise try to spin up extra replica pods/PVCs you likely don't want
  master:
    persistence:
      enabled: true
      storageClass: longhorn
      size: 2Gi

supersetNode:
  replicaCount: 1

supersetWorker:
  replicaCount: 1

supersetCeleryBeat:
  enabled: true   # NOTE: chart default is `false` — required for alerts/reports scheduling
```

⚠️ **Secrets in this file (`SECRET_KEY`, Postgres password, admin password) are
plaintext.** Fine for a lab. Rotate all three before treating this as anything
beyond disposable/local.

---

## Deploy from scratch

```bash
# 1. Namespace
kubectl create namespace pms-superset

# 2. Validate before touching the cluster
helm lint .
helm template superset . -n pms-superset > rendered.yaml
echo $?                                    # want 0
grep -n "psycopg2" rendered.yaml           # confirm bootstrap fix is templated
grep -n "storageClass" rendered.yaml       # want `longhorn` x2
grep -E '^kind:|^  name:' rendered.yaml | head -80

# 3. Install
helm install superset . -n pms-superset

# 4. Watch it come up
kubectl get pods -n pms-superset -w
```

Expect this steady-state once healthy:

```
pod/superset-...              1/1   Running
pod/superset-celerybeat-...   1/1   Running
pod/superset-init-db-...      0/1   Completed     <- Job pod, expected to show 0/1 Ready once done
pod/superset-postgresql-0     1/1   Running
pod/superset-redis-master-0   1/1   Running
pod/superset-worker-...       1/1   Running
```

---

## Access the app

```bash
kubectl get nodes -o wide     # grab any node's INTERNAL-IP
```

Browse to: `http://<NODE-INTERNAL-IP>:30088`

Login: `admin` / `admin` (or whatever you set in `init.adminUser`)

**First thing to do after logging in:** change the admin password.

---

## Troubleshooting

### `ModuleNotFoundError: No module named 'psycopg2'`
The `bootstrapScript` isn't installing the driver into the right place.
Confirm what's actually live in the cluster (not just in your local file):

```bash
kubectl get secret superset-config -n pms-superset \
  -o jsonpath='{.data.superset_bootstrap\.sh}' | base64 -d
```

Should show:
```bash
uv pip install --python /app/.venv/bin/python psycopg2-binary
```

If it doesn't, your `values.yaml` edit didn't make it into the last
`helm install`/`upgrade` — re-check, re-run.

### `psycopg2.OperationalError: ... password authentication failed`
Postgres only sets its password **once**, at first PVC initialization. If
you've changed `postgresql.auth.password` in `values.yaml` after an earlier
install, the running Postgres still has the *old* password baked into its
data volume. `helm upgrade`/`helm install` will not fix this — the PVC has to
be wiped so Postgres reinitializes.

```bash
kubectl get pvc -n pms-superset
helm uninstall superset -n pms-superset
kubectl delete pvc --all -n pms-superset   # ⚠️ destroys all data — fine for lab, NOT for prod
kubectl get pvc -n pms-superset            # confirm empty
helm install superset . -n pms-superset
```

### Init job stuck in `Error` across multiple pods
The `init` Job re-creates a new pod on every `helm upgrade`/reinstall attempt
if the previous one failed. Old failed pods can pile up and aren't always
cleaned up by `helm uninstall`. Clear them explicitly:

```bash
kubectl delete job superset-init-db -n pms-superset --ignore-not-found
```

To see only the **latest** attempt's logs (not a confusing concatenation of
every historical attempt):

```bash
POD=$(kubectl get pods -n pms-superset -l job-name=superset-init-db \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')
kubectl logs -n pms-superset "$POD"
```

### `helm lint` error: `cannot overwrite table with non table for service.nodePort`
This chart expects `service.nodePort` as a nested map, not a flat integer:

```yaml
# Wrong
service:
  nodePort: 30088

# Right
service:
  nodePort:
    http: 30088
```

### Unexpected `superset-redis-replicas-0` pod stuck `Pending`
`redis.architecture` defaults to `replication` if left unset/commented out in
`values.yaml`, spinning up extra replica pods (and PVCs) you likely don't
want for a single-node lab setup. Explicitly set:

```yaml
redis:
  architecture: standalone
```

---

## Debugging inside the actual image

Useful when you need to check what's available inside the exact Superset
image without going through Helm/the running deployment:

```bash
kubectl run superset-debug --rm -it --restart=Never \
  --image=apachesuperset.docker.scarf.sh/apache/superset:6.1.0 \
  -n pms-superset -- sh -c "ls /app/.venv/bin/ | grep -i pip; which uv"
```

---

## Full command reference

```bash
# Validation
helm lint .
helm template superset . -n pms-superset > rendered.yaml
echo $?
grep -n "psycopg2" rendered.yaml
grep -n "storageClass" rendered.yaml
grep -n "nodePort" rendered.yaml
grep -E '^kind:|^  name:' rendered.yaml | head -80

# Install / uninstall
kubectl create namespace pms-superset
helm install superset . -n pms-superset
helm uninstall superset -n pms-superset

# Inspect state
kubectl get all -n pms-superset
kubectl get pvc -n pms-superset
kubectl get pods -n pms-superset -w
kubectl describe pod -n pms-superset <pod-name>
kubectl describe deployment superset -n pms-superset

# Logs
kubectl logs -n pms-superset <pod-name>
kubectl logs -n pms-superset -l job-name=superset-init-db --tail=50
POD=$(kubectl get pods -n pms-superset -l job-name=superset-init-db \
  --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
kubectl logs -n pms-superset "$POD"

# Confirm live config matches values.yaml
kubectl get secret superset-config -n pms-superset \
  -o jsonpath='{.data.superset_bootstrap\.sh}' | base64 -d

# Cleanup
kubectl delete job superset-init-db -n pms-superset --ignore-not-found
kubectl delete pvc --all -n pms-superset   # ⚠️ destroys data

# Access
kubectl get nodes -o wide
# -> http://<NODE-INTERNAL-IP>:30088
```
