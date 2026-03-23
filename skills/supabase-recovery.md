---
name: supabase-recovery
description: Recover from a Supabase Supavisor circuit breaker being open due to too many authentication errors. STOP all connections immediately before doing anything else.
---

# /supabase-recovery

Recover from: `FATAL: Circuit breaker open: Too many authentication errors`

**STOP ALL CONNECTION ATTEMPTS FIRST.** Every failed connection resets the circuit breaker timer. Do not test, ping, or query the DB until Step 8.

---

## Step 1 — Scale all workloads to zero immediately

```bash
microk8s kubectl scale deployment --all --replicas=0 -n ordomatics-{ENV}
```

Verify no pods are running:
```bash
microk8s kubectl get pods -n ordomatics-{ENV}
# Expected: No resources found
```

## Step 2 — Pause the Supabase project

Via MCP:
```
mcp__supabase__pause_project({ project_id: "{SUPABASE_PROJECT_ID}" })
```

## Step 3 — Wait for INACTIVE status

```
mcp__supabase__get_project({ id: "{SUPABASE_PROJECT_ID}" })
# Wait until status == "INACTIVE"
```

This takes 1-3 minutes.

## Step 4 — Restore the project

```
mcp__supabase__restore_project({ project_id: "{SUPABASE_PROJECT_ID}" })
```

Wait for status to return to `"ACTIVE_HEALTHY"`.

## Step 5 — Reset the DB password

In the Supabase dashboard: Settings → Database → Reset database password.

Update `.env`:
```
SUPABASE_{ENV}_PASSWORD=<new-password>
```

## Step 6 — Update the K8s secret

```bash
microk8s kubectl create secret generic ordomatics-{ENV}-secrets \
  -n ordomatics-{ENV} \
  --from-literal=DB_PASSWORD="<new-password>" \
  ... (all other keys) \
  --dry-run=client -o yaml | microk8s kubectl apply -f -
```

## Step 7 — Wait 5 minutes with ZERO connection attempts

Do not run any queries, psql, or curl to the app. Every attempt (even failed ones) resets the breaker timer.

## Step 8 — Test once with psql

```bash
PGPASSWORD=<new-password> psql \
  "postgresql://odoo.<ref>.pooler.supabase.com:6543/postgres?sslmode=require"
```

- If it connects: proceed to Step 9.
- If still blocked: wait another 10 minutes. Do NOT keep retrying.

## Step 9 — Scale workloads back up

```bash
microk8s kubectl scale deployment ordomatics-{ENV} --replicas=1 -n ordomatics-{ENV}
microk8s kubectl scale deployment ordomatics-{ENV}-longpolling --replicas=1 -n ordomatics-{ENV}
```

---

## Gotchas

- **Direct host `db.{ref}.supabase.co` is IPv6-only** — unreachable from IPv4-only machines. Use the pooler host `odoo.<ref>.pooler.supabase.com` instead.
- **Port 5432 (session mode) and port 6543 (transaction mode) share the same circuit breaker.** Switching ports does not help.
- **Every failed attempt resets the timer** — including `pg_isready`, `psql`, health check probes, and app connection pool retries.
- **Pause/restore resets the Supavisor process state** — this is the only reliable way to clear the circuit breaker without waiting 30+ minutes.
