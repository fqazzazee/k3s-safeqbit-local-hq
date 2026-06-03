# Netbox: Migrating from Docker/Portainer to k3s

**Date:** 2026-05-15  
**Cluster:** safeqbit-local-hq

---

## What Changed and Why

Netbox was running on a dedicated Docker host (`10.10.12.34`) deployed as a Portainer stack. This meant:

- A separate VM/host to maintain and keep patched
- No GitOps - config lived in Portainer's UI, not in version control
- Hand-rolled `postgres:16-alpine` container with a plaintext password in the Portainer `.env` file
- No application-consistent database backups
- Two Redis containers configured by hand with no persistence guarantees on the cache vs queue split

Moving to k3s gives us GitOps (the whole stack is now in Git), CNPG for Postgres (same pattern as Authentik and Grafana), and Velero biweekly backups to B2.

---

## Architecture: Old vs New

### Old Setup (Portainer stack)

```
Docker host: 10.10.12.34 (macvlan IP)
├── postgres:16-alpine          ← hand-rolled, password in .env
├── redis:7-alpine              ← task queue, persistent volume
├── redis:7-alpine              ← cache, LRU, no volume
├── netboxcommunity/netbox:v4.2 ← web (port 8080)
├── netboxcommunity/netbox:v4.2 ← rqworker
└── netboxcommunity/netbox:v4.2 ← housekeeping (cron inside container)
```

- Backups: none (Velero doesn't know about the Docker host)
- Credentials: plaintext in Portainer's `.env` file
- Upgrades: manual docker pull + recreate in Portainer UI

### New Setup (k3s)

```
┌─────────────────────────────────────────────────────────────────┐
│                        netbox namespace                         │
│                                                                 │
│  ┌─── ──────────────────────┐   ┌───────────────────────────┐   │
│  │   CNPG Cluster CR        │   │  Deployment: netbox      │    │
│  │   (netbox-cnpg)          │   │  (web / gunicorn via Unit)│   │
│  │                          │   └──────────────┬────────────┘   │
│  │  ┌────────────────────┐  │                  │ DB_HOST=       │
│  │  │  netbox-cnpg-1     │◄─┼──────────────────┘ netbox-cnpg-rw │
│  │  │  (Postgres 16)     │  │                                   │
│  │  └────────┬───────────┘  │   ┌───────────────────────────┐   │
│  │           │              │   │  Deployment: netbox-worker│   │
│  │  ┌────────▼───────────┐  │   │  (rqworker)               │   │
│  │  │  Longhorn PVC 5Gi  │  │   └───────────────────────────┘   │
│  │  └────────────────────┘  │                                   │
│  │                          │   ┌───────────────────────────┐   │
│  │  Services (auto):        │   │  CronJob: housekeeping    │   │
│  │  • netbox-cnpg-rw        │   │  (daily 00:30 UTC)        │   │
│  │  • netbox-cnpg-ro        │   └───────────────────────────┘   │
│  │  • netbox-cnpg-r         │                                   │
│  │                          │   ┌───────────────────────────┐   │
│  │  Secret (auto):          │   │  Deployment: redis-tasks  │   │
│  │  • netbox-cnpg-app       │   │  redis:7-alpine           │   │
│  └─────────────────────── ──┘   │  Longhorn PVC 1Gi         │   │
│                                 │  --appendonly yes         │   │
│  ┌──────────────────────-───┐   └───────────────────────────┘   │
│  │  SealedSecret            │                                   │
│  │  (netbox-credentials)    │   ┌───────────────────────────┐   │
│  │  • redis-password        │   │  Deployment: redis-cache  │   │
│  │  • redis-cache-password  │   │  redis:7-alpine           │   │
│  │  • secret-key            │   │  --maxmemory 512mb        │   │
│  │  • superuser-*           │   │  --maxmemory-policy       │   │
│  └─────────────────────── ──┘   │    allkeys-lru            │   │
│                                 │  no PVC (ephemeral)       │   │
│  ┌────────────────────── ───┐   └───────────────────────────┘   │
│  │  NFS PVCs (nfs-truenas)  │                                   │
│  │  • netbox-media   10Gi   │   ┌───────────────────────────┐   │
│  │  • netbox-reports  1Gi   │   │  Ingress                  │   │
│  │  • netbox-scripts  1Gi   │   │  netbox.local.safeqbit.com│   │
│  │  (RWX - shared by web    │   │  cert-manager letsencrypt │   │
│  │   and worker)            │   └───────────────────────────┘   │
│  └───────────────────── ────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions

### Why raw manifests instead of a Helm chart

The only maintained community Netbox Helm chart (`bootc/netbox`) has APP VERSION `v3.2.8` at its latest release (chart 4.1.1). Netbox `v4.2` is a major version with breaking changes from 3.x. Using the chart with an overridden image tag would risk subtle incompatibilities in the generated configs.

Raw manifests give full control and are a direct 1:1 port of the Portainer stack. The stack is simple enough that a chart adds more complexity than it removes.

### Why two separate Redis deployments

Netbox uses Redis for two fundamentally different purposes with different durability requirements:

| Instance | Service name | Purpose | Persistence | Memory |
|---|---|---|---|---|
| Task queue | `netbox-redis-tasks` | RQ job queue - webhooks, custom scripts | Yes (Longhorn 1Gi, `--appendonly yes`) | Default |
| Cache | `netbox-redis-cache` | Object cache - safe to lose | No PVC | 512mb LRU cap |

If you used a single Redis for both and it restarted, you'd lose both cached objects (fine) **and** queued jobs (not fine - webhooks silently dropped). Keeping them separate means a cache restart is harmless.

### Why NFS RWX for media/reports/scripts

The web pod and the worker pod are separate Kubernetes Deployments - separate pods that can land on different nodes. Longhorn PVCs are `ReadWriteOnce` (one node at a time), which would force co-scheduling and defeat the purpose of having separate pods.

`nfs-truenas` provides `ReadWriteMany`, so both pods mount the same NFS share simultaneously. This matches the Docker Compose behavior where all three containers shared the same Docker volume.

### SKIP_SUPERUSER behavior

The `netboxcommunity/netbox` image runs `create_superuser.py` at startup when `SKIP_SUPERUSER=false`. This script checks if the superuser already exists - if it does, it skips creation and does not overwrite the existing password. Setting `SKIP_SUPERUSER=false` on the web pod is safe even after restoring an existing database. The worker and housekeeping containers set `SKIP_SUPERUSER=true` because they have no business creating admin users.

---

## Migration Runbook

### Pre-requisites

- Docker host accessible via SSH from the k3s machine
- `pg_dump` available inside the Docker postgres container
- `kubeseal` binary at `/tmp/kubeseal` (v0.28.0 matching controller 0.36.6)

### Step 1 - Seal the credentials

Collect from the running Portainer stack:
- Redis task queue password
- Redis cache password
- Netbox `SECRET_KEY`
- Superuser email, password, API token

```bash
kubectl create secret generic netbox-credentials \
  --namespace netbox \
  --dry-run=client \
  --from-literal=redis-password='<value>' \
  --from-literal=redis-cache-password='<value>' \
  --from-literal=secret-key='<value>' \
  --from-literal=superuser-email='<value>' \
  --from-literal=superuser-password='<value>' \
  --from-literal=superuser-api-token='<value>' \
  -o yaml | \
/tmp/kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  -o yaml > apps/safeqbit-local-hq/netbox/02-sealed-secrets.yaml
```

**Important:** `SECRET_KEY` must match the old instance exactly. Netbox uses it to sign session cookies and tokens. A changed `SECRET_KEY` invalidates all existing sessions on next login (not catastrophic, but disruptive).

### Step 2 - Push and wait for initial boot

```bash
git add apps/safeqbit-local-hq/netbox/ \
        infrastructure/safeqbit-local-hq/configs/velero-schedule-netbox.yaml \
        infrastructure/safeqbit-local-hq/configs/kustomization.yaml \
        apps/safeqbit-local-hq/kustomization.yaml
git commit -m "apps: add Netbox migration to k3s"
git push origin main

# Trigger Flux immediately rather than waiting up to 10m
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite

# Wait for CNPG to initialise its database
until kubectl get cluster netbox-cnpg -n netbox \
  -o jsonpath='{.status.readyInstances}' | grep -q "1"; do sleep 5; done
echo "CNPG ready"

# Wait for the web pod to pass its readiness probe
kubectl rollout status deploy/netbox -n netbox --timeout=5m
```

At this point Netbox has booted against an empty database and applied the full schema via Django migrations. This is expected - we will wipe and restore in the next step.

### Step 3 - Scale to zero before restore

```bash
kubectl scale deploy netbox netbox-worker -n netbox --replicas=0
```

This prevents Netbox from writing new sessions or changes to the database while we overwrite it. Always do this before a restore - active writes during a restore corrupt the data.

### Step 4 - Dump from Docker host

On the Docker host, find the Postgres container name:

```bash
docker ps --format '{{.Names}}' | grep -i postgres
```

Dump the database with `--clean --if-exists`:

```bash
docker exec <postgres-container> \
  pg_dump -U netbox --clean --if-exists netbox > /tmp/netbox.sql
```

**`--clean --if-exists`** - generates `DROP ... IF EXISTS` before every `CREATE`. This makes the restore idempotent: when applied on top of an existing schema (which Netbox already created during initial boot), each object is dropped and recreated cleanly instead of erroring on "already exists".

Copy the dump to the k3s machine:

```bash
scp user@10.10.12.34:/tmp/netbox.sql /tmp/netbox.sql
```

### Step 5 - Wipe the fresh schema and restore

```bash
# Drop the schema created by Netbox's initial boot
kubectl exec -n netbox netbox-cnpg-1 -- psql -U postgres netbox \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public; GRANT ALL ON SCHEMA public TO netbox;"

# Restore the dump
kubectl exec -i netbox-cnpg-1 -n netbox -- psql -U postgres netbox < /tmp/netbox.sql
```

**Why drop the schema first?** The `--clean` flag in pg_dump emits `DROP` statements for every object, but it emits them in dependency order for individual objects - not as a single schema wipe. On a database that was initialised by Django migrations, there are circular dependencies and extension objects that `--clean` alone can't resolve cleanly. Dropping the whole public schema first gives pg_restore a clean slate.

**Why `psql -U postgres` instead of `-U netbox`?** Inside the CNPG pod, the `postgres` superuser can connect via the Unix socket (`/controller/run/.s.PGSQL.5432`) using peer authentication. The `netbox` app user requires password authentication via TCP. Using `postgres` avoids needing to pass the password, and the CNPG pod doesn't expose psql to the `netbox` user via peer auth. For the `GRANT` statement, we need superuser privileges anyway.

### Step 6 - Scale back up

```bash
kubectl scale deploy netbox netbox-worker -n netbox --replicas=1

# Wait for web to be healthy
kubectl rollout status deploy/netbox -n netbox --timeout=5m
```

Check the logs to confirm Netbox found the existing superuser (not creating a new one):

```bash
kubectl logs -n netbox -l app.kubernetes.io/component=web --tail=10
# Expect: "💡 Superuser Username: admin, E-Mail: fadi@safeqbit.com"
# and:    "✅ Initialisation is done."
```

### Step 7 - Sync media/reports/scripts (if applicable)

The NFS PVCs mount to these paths inside the running pod:

```
/opt/netbox/netbox/media    → 10.10.10.5:/mnt/nvme2tb/k8s/pvs/netbox-netbox-media-${.PV.name}
/opt/netbox/netbox/reports  → 10.10.10.5:/mnt/nvme2tb/k8s/pvs/netbox-netbox-reports-${.PV.name}
/opt/netbox/netbox/scripts  → 10.10.10.5:/mnt/nvme2tb/k8s/pvs/netbox-netbox-scripts-${.PV.name}
```

**Note on `${.PV.name}` in the path:** The `nfs-subdir-external-provisioner` uses a `pathPattern` template to generate subdirectory names. The template variable `${.PV.name}` is not expanded in the PV spec or `df` output - it appears literally. But the NFS server created that directory with this exact literal name, and it works. It's a quirk of this provisioner version's metadata storage.

If the Docker volumes had content, rsync from the Docker host to TrueNAS via SSH (single-quote the path to prevent shell expansion of `${...}`):

```bash
# Run on Docker host
rsync -av /var/lib/docker/volumes/<media_volume>/_data/ \
  user@10.10.10.5:'/mnt/nvme2tb/k8s/pvs/netbox-netbox-media-${.PV.name}/'
```

In this migration the Docker volumes were empty - no files to copy.

---

## Health Probe Fix: HTTP → TCP

The initial manifests used HTTP GET probes on `/`:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
```

This caused the pod to never become Ready. The kube-probe sends HTTP requests with a `kube-probe/1.x` User-Agent and no `Host` header matching `ALLOWED_HOSTS`. Django returns HTTP 400 (Bad Request) for any request whose `Host` header doesn't match - even health checks. The pod logs showed:

```
GET / HTTP/1.1" 400 156 "-" "kube-probe/1.35"
```

**Fix:** Switch to TCP socket probes. These just check that the port is accepting connections - they don't send an HTTP request at all, so Django's host validation is never triggered.

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
livenessProbe:
  tcpSocket:
    port: 8080
```

The same principle applies to any Django or other ALLOWED_HOSTS-enforcing app running behind a k8s ingress. Alternatives would be adding `127.0.0.1` or `*` to `ALLOWED_HOSTS`, but that weakens the security boundary unnecessarily when a TCP check is sufficient.

---

## Redis Health Probes

The Redis containers also needed a non-obvious probe fix. The initial config used:

```yaml
livenessProbe:
  exec:
    command: ["redis-cli", "ping"]
```

But `redis-cli ping` against a password-protected Redis returns `-NOAUTH Authentication required` - not `PONG`. The probe would succeed (exit 0) or fail depending on the redis-cli version. TCP socket is the correct approach here too:

```yaml
livenessProbe:
  tcpSocket:
    port: 6379
```

This confirms the Redis process is accepting connections, which is all the liveness probe needs to know.

---

## Key Files

```
apps/safeqbit-local-hq/netbox/
├── 01-namespace.yaml        Namespace with backup.policy/velero label
├── 02-sealed-secrets.yaml   SealedSecret: redis passwords, SECRET_KEY, superuser creds
├── 03-cnpg-cluster.yaml     CNPG Cluster: postgres 16, instances: 1, Longhorn 5Gi
├── 04-redis.yaml            Two Redis Deployments + PVC (tasks) + two Services
├── 05-pvcs.yaml             media/reports/scripts on nfs-truenas (ReadWriteMany)
├── 06-netbox-web.yaml       Web Deployment + ClusterIP Service
├── 07-worker.yaml           rqworker Deployment (same image, different command)
├── 08-housekeeping.yaml     CronJob, daily 00:30 UTC, housekeeping.sh
└── 09-ingress.yaml          netbox.local.safeqbit.com, letsencrypt-prod

infrastructure/safeqbit-local-hq/configs/
└── velero-schedule-netbox.yaml   Biweekly 1st & 15th at 03:00 UTC, 6-month retention
```

---

## Useful Commands

```bash
# Check all Netbox pods
kubectl get pods -n netbox

# Tail web logs
kubectl logs -n netbox -l app.kubernetes.io/component=web -f

# Tail worker logs
kubectl logs -n netbox -l app.kubernetes.io/component=worker -f

# Connect to the Netbox Postgres DB
kubectl exec -it -n netbox netbox-cnpg-1 -- psql -U postgres netbox

# Check CNPG cluster health
kubectl get cluster -n netbox netbox-cnpg

# Verify Redis task queue is responding (from any pod in the namespace)
kubectl exec -n netbox deploy/netbox -- \
  redis-cli -h netbox-redis-tasks -a '<password>' ping

# Trigger a manual housekeeping run
kubectl create job --from=cronjob/netbox-housekeeping netbox-housekeeping-manual -n netbox

# Check Velero schedule
kubectl get schedule netbox-biweekly -n velero
```

---

## Lessons Learned

**Django's `ALLOWED_HOSTS` breaks HTTP health probes.** Kubernetes sends liveness/readiness probes without the Host header the app expects. Any Django app (Netbox, Wagtail, etc.) will return 400 on probe requests unless you add `127.0.0.1` to `ALLOWED_HOSTS` or switch to TCP socket probes. TCP socket is simpler and has no security implications.

**`redis-cli ping` fails silently on password-protected Redis.** Always use `tcpSocket` probes for password-protected Redis. The exec probe approach only works for unauthenticated Redis.

**Scale to zero before a database restore.** If the application is running while you restore, it will write new sessions or state entries concurrently with the restore, corrupting the result. Scale web and worker to 0 first, restore, then scale back up.

**The Docker volumes were empty.** Netbox stores uploaded images (device photos, rack diagrams) in `media/`. If you actively use that feature, syncing `media/` is critical. In this deployment it was unused - the migration was DB-only.

**`POSTGRES_PASSWORD` from the Docker stack is not needed in k3s.** CNPG generates its own password and stores it in `netbox-cnpg-app`. The old Postgres password is only needed for the one-time pg_dump command on the Docker host. Do not commit it anywhere.
