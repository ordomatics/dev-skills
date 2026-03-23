---
name: odoo-env-reset
description: Full wipe and redeploy of an Ordomatics K8s environment (dev/test/staging/prod). Encodes all gotchas from repeated wipe+redeploy sessions.
---

# /odoo-env-reset

Full namespace wipe and redeploy for an Ordomatics K8s environment via ArgoCD.

Ask the user which environment to reset if not specified: `dev`, `test`, `staging`, or `prod`.

---

## Step 1 — Clear ArgoCD finalizers on any existing jobs

Jobs created by ArgoCD hooks have a finalizer that blocks namespace deletion.

```bash
for job in $(microk8s kubectl get jobs -n ordomatics-{ENV} -o name 2>/dev/null); do
  microk8s kubectl patch $job -n ordomatics-{ENV} \
    -p '{"metadata":{"finalizers":null}}' --type=merge
done
```

## Step 2 — Delete the ArgoCD Application and namespace

```bash
microk8s kubectl delete application ordomatics-{ENV} -n argocd --ignore-not-found
microk8s kubectl delete ns ordomatics-{ENV} --grace-period=0 --force
```

Verify it's gone:
```bash
microk8s kubectl get ns ordomatics-{ENV}
# Expected: Error from server (NotFound)
```

## Step 3 — Recreate the namespace and secrets

```bash
microk8s kubectl create ns ordomatics-{ENV}
```

### gitlab-registry (image pull secret)

```bash
source /home/moctardiallo/projects/ordomatics/odoo/.env

microk8s kubectl create secret docker-registry gitlab-registry \
  -n ordomatics-{ENV} \
  --docker-server=registry.gitlab.com \
  --docker-username=moctardiallo \
  --docker-password="${GITLAB_ACCESS_TOKEN}"
```

### ordomatics-{ENV}-secrets

**CRITICAL: ADMIN_PASSWD must be set — missing it causes an immediate pod crash.**

For **dev** (in-cluster postgres, password is literal "odoo"):
```bash
microk8s kubectl create secret generic ordomatics-dev-secrets \
  -n ordomatics-dev \
  --from-literal=DB_HOST=ordomatics-dev-postgres \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_USER=odoo \
  --from-literal=DB_PASSWORD=odoo \
  --from-literal=DB_NAME=odoo \
  --from-literal=ADMIN_PASSWD="${ODOO_MASTER_PASSWORD}" \
  --from-literal=DB_URI="" \
  --from-literal=ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY:-}" \
  --from-literal=OPENAI_API_KEY="${OPENAI_API_KEY:-}" \
  --from-literal=WAVE_CHECKOUT_API_KEY="" \
  --from-literal=WAVE_PAYOUT_API_KEY="" \
  --from-literal=WAVE_BALANCE_API_KEY="" \
  --from-literal=WAVE_PAYMENT_RECEIVED_SHARED_SECRET="" \
  --from-literal=META_VERIFY_TOKEN="" \
  --from-literal=META_ACCESS_TOKEN="" \
  --from-literal=WHATSAPP_PHONE_NUMBER="" \
  --from-literal=WHATSAPP_PHONE_NUMBER_ID="" \
  --from-literal=WHATSAPP_BUSINESS_ACCOUNT_ID="" \
  --from-literal=WHATSAPP_FLOW_PRIVATE_KEY="" \
  --from-literal=DULAYNY_API_KEY="" \
  --from-literal=AGENTIC_API_KEY=""
```

For **test/staging/prod**: use the corresponding `SUPABASE_{ENV}_*` vars from `.env` for DB credentials.

## Step 4 — Restart ArgoCD repo server (preemptively)

The repo server crash-loops on restart due to a `copyutil` init container "Already exists" bug.
Always restart it before applying the ArgoCD app to avoid stale cache and crash-loops.

```bash
microk8s kubectl rollout restart deployment argocd-repo-server -n argocd
# Wait for it to be Running
microk8s kubectl get pods -n argocd | grep repo
```

## Step 5 — Apply the ArgoCD Application

```bash
microk8s kubectl apply -f /home/moctardiallo/projects/ordomatics/helm/application-{ENV}.yaml -n argocd
```

## Step 6 — Trigger ArgoCD hard refresh + sync

Port-forward, authenticate, and sync via REST API (argocd CLI may not be available):

```bash
microk8s kubectl port-forward svc/argocd-server -n argocd 8889:80 &>/dev/null &
sleep 2

PASS=$(microk8s kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)
TOKEN=$(curl -sk -X POST http://localhost:8889/api/v1/session \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"admin\",\"password\":\"$PASS\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Hard refresh (invalidates stale manifest cache)
curl -sk "http://localhost:8889/api/v1/applications/ordomatics-{ENV}?refresh=hard" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null
sleep 3

# Sync with prune
curl -sk -X POST "http://localhost:8889/api/v1/applications/ordomatics-{ENV}/sync" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"prune":true}' -o /dev/null

kill $(lsof -ti:8889) 2>/dev/null
```

## Step 7 — Import Odoo image into MicroK8s containerd (dev only)

MicroK8s containerd is separate from the Docker daemon. After a namespace wipe,
`imagePullPolicy: IfNotPresent` won't pull from registry — import from local Docker instead.

```bash
# Check if image is already present
/snap/microk8s/current/bin/ctr \
  --address /var/snap/microk8s/common/run/containerd.sock \
  --namespace k8s.io images ls | grep "helm/odoo:" | grep -v postgres

# If missing, import from Docker (takes ~2-3 min for 4.3GB image)
docker save registry.gitlab.com/ordomatics/helm/odoo:latest | \
  /snap/microk8s/current/bin/ctr \
  --address /var/snap/microk8s/common/run/containerd.sock \
  --namespace k8s.io images import -
```

**Gotcha:** The system `ctr` binary fails with "unknown service containerd.services.streaming.v1.Streaming".
Always use MicroK8s's own binary at `/snap/microk8s/current/bin/ctr`.

## Step 8 — Wait for pods and verify

```bash
microk8s kubectl get pods -n ordomatics-{ENV}
```

For **dev**: Odoo will be `0/1 Running` for 5-8 minutes while it runs module setup
(`skipModuleSetup=false` — Odoo initializes the DB itself on first boot).
The startup probe allows 10 minutes of failure — do not intervene during this window.

```bash
# Tail logs to confirm module setup is progressing (not erroring)
microk8s kubectl logs -f <odoo-pod> -n ordomatics-{ENV} 2>&1 | grep "Loading module"
```

Expected completion signal:
```
GET /web/health HTTP/1.1" 200
```

Final check:
```bash
curl http://dev.ordomatics.local/web/health
# Expected: {"status": "pass"}
```

---

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | `gitlab-registry` secret missing or wrong token | Recreate the secret |
| Pod crash at startup | `ADMIN_PASSWD` not in secrets | Add it to the secret |
| Namespace stuck Terminating | ArgoCD job finalizer | Patch `finalizers: null` on jobs (Step 1) |
| ArgoCD `ComparisonError: connection refused` | Repo server crash-loop | `kubectl rollout restart deployment argocd-repo-server -n argocd` |
| Pods `ContainerCreating` with no events for >5 min | Odoo image not in containerd | Import via microk8s ctr (Step 7) |
| ArgoCD shows stale manifests after commit | Cache not invalidated | Hard refresh before sync (Step 6) |
