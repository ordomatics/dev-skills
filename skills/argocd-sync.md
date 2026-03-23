---
name: argocd-sync
description: Trigger an ArgoCD application sync via REST API. Use when the argocd CLI is not available or when a hard refresh + forced sync is needed.
---

# /argocd-sync

Trigger a hard refresh and sync for an ArgoCD application via the REST API.

Ask the user which app/environment to sync if not specified.

---

## Full sequence

```bash
# 1. Kill any existing port-forward on 8889, then start a new one
kill $(lsof -ti:8889) 2>/dev/null
microk8s kubectl port-forward svc/argocd-server -n argocd 8889:80 &>/dev/null &
sleep 2

# 2. Get the admin password
PASS=$(microk8s kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)

# 3. Authenticate — endpoint is /api/v1/session (SINGULAR, not "sessions")
TOKEN=$(curl -sk -X POST http://localhost:8889/api/v1/session \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"admin\",\"password\":\"$PASS\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# 4. Hard refresh (clears stale manifest cache from git)
curl -sk "http://localhost:8889/api/v1/applications/ordomatics-{ENV}?refresh=hard" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null
sleep 3

# 5. Sync with prune
curl -sk -X POST "http://localhost:8889/api/v1/applications/ordomatics-{ENV}/sync" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prune":true}' -o /dev/null

# 6. Kill port-forward
kill $(lsof -ti:8889) 2>/dev/null
```

## Check sync status

```bash
microk8s kubectl port-forward svc/argocd-server -n argocd 8889:80 &>/dev/null &
sleep 2
PASS=$(microk8s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
TOKEN=$(curl -sk -X POST http://localhost:8889/api/v1/session -H "Content-Type: application/json" \
  -d "{\"username\":\"admin\",\"password\":\"$PASS\"}" | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

curl -sk "http://localhost:8889/api/v1/applications/ordomatics-{ENV}" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "
import sys,json
d=json.load(sys.stdin)
s=d.get('status',{})
print('Sync:', s.get('sync',{}).get('status'))
print('Health:', s.get('health',{}).get('status'))
print('Op phase:', s.get('operationState',{}).get('phase'))
print('Message:', s.get('operationState',{}).get('message','')[:120])
"
kill $(lsof -ti:8889) 2>/dev/null
```

---

## Gotchas

- **`/api/v1/sessions` (plural) → 404.** The correct endpoint is `/api/v1/session` (singular).
- **ArgoCD serves HTTP on port 80**, not HTTPS — port-forward to 80, not 443.
- **Tokens expire quickly** — get token and make the request in one shell session without sleeping in between.
- **After namespace delete**, ArgoCD shows `Synced` from stale cache. Always hard-refresh before sync.
- **If `ComparisonError: connection refused`** — the repo server is crash-looping. Fix: `kubectl rollout restart deployment argocd-repo-server -n argocd` and wait for it to be Running before retrying.
