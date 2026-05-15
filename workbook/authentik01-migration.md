# Authentik01: Migrating from Docker/Portainer to k3s

**Date:** 2026-05-15  
**Cluster:** safeqbit-local-hq  
**Source host:** authentik01.local.safeqbit.com (macvlan 10.10.12.29)

---

## What Changed and Why

Authentik01 was running as a Portainer stack on a standalone Docker host with a macvlan IP.
Moving it to k3s gives GitOps, CNPG for Postgres, and Velero backups — the same pattern used
for the primary `authentik` namespace migrated earlier.

### Old Setup (Portainer stack)

```
Docker host: 10.10.12.29 (macvlan IP)
├── postgres:16-alpine          ← hand-rolled, password in stack.env
├── authentik-server            ← ghcr.io/goauthentik/server:2025.10.3
└── authentik-worker            ← ghcr.io/goauthentik/server:2025.10.3
```

Media at: `/var/lib/docker/volumes/containers_vol/authentik/media`  
Templates at: `/var/lib/docker/volumes/containers_vol/authentik/custom-templates`

### New Setup (k3s)

```
Namespace: authentik01
├── CNPG Cluster: authentik01-cnpg (Postgres 16, instances: 2, Longhorn 10Gi)
├── HelmRelease: authentik01 (chart: authentik 2025.10.3, redis subchart enabled)
├── NFS PVCs: authentik01-media (5Gi), authentik01-templates (1Gi)
└── Ingress: authentik01.local.safeqbit.com (letsencrypt-prod)
```

---

## Architecture Notes

**Why `instances: 2` for CNPG:** Authentik01 is an SSO backend. The same reasoning as the
primary `authentik` cluster — if the database is down, anything this instance protects
breaks. A hot standby cuts failover time from ~60s (pod reschedule) to ~10–30s (CNPG
promotion).

**Why HelmRelease name is `authentik01`:** The chart uses the release name as the prefix for
generated services. Naming it `authentik01` produces `authentik01-server` (the service the
ingress routes to), which is unambiguous when both `authentik` and `authentik01` exist in
the cluster.

**Why `redis.enabled: true`:** The Portainer stack didn't include a Redis container
explicitly, but Authentik requires Redis. The Helm chart ships a Redis subchart — enabling
it is the simplest approach and matches the primary `authentik` setup.

**Backup schedule staggered:** CNPG ScheduledBackup runs at 02:00 UTC (primary `authentik-cnpg`
runs at 01:00 UTC) to avoid simultaneous snapshot I/O on Longhorn.

---

## Migration Runbook

### Pre-requisites

- Docker host accessible via SSH
- `pg_dump` available inside the Docker postgres container
- `kubeseal` binary at `/tmp/kubeseal` (v0.28.0, matching controller 0.36.6)
- The `AUTHENTIK_SECRET_KEY` value from `stack.env` on the Docker host

### Step 1 — Seal the credentials

**Do this before pushing to Flux.** The HelmRelease won't boot until the Secret exists.

SSH to the Docker host and retrieve `AUTHENTIK_SECRET_KEY` from the stack env:

```bash
# On the Docker host — find the running stack env
cat /path/to/authentik/stack.env | grep AUTHENTIK_SECRET_KEY
```

Then on the k3s machine, seal it:

```bash
kubectl create secret generic authentik01-credentials \
  --namespace authentik01 \
  --dry-run=client \
  --from-literal=AUTHENTIK_SECRET_KEY='<value>' \
  -o yaml | \
/tmp/kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  -o yaml > apps/safeqbit-local-hq/authentik01/02-sealed-secrets.yaml
```

**Critical:** `AUTHENTIK_SECRET_KEY` must match the running Docker stack exactly.
Authentik uses it to sign sessions and tokens — changing it invalidates all active sessions.

### Step 2 — Push and wait for initial boot

```bash
git add apps/safeqbit-local-hq/authentik01/ \
        infrastructure/safeqbit-local-hq/configs/velero-schedule-authentik01.yaml \
        infrastructure/safeqbit-local-hq/configs/kustomization.yaml \
        apps/safeqbit-local-hq/kustomization.yaml
git commit -m "apps: add Authentik01 migration to k3s"
git push origin main

# Trigger Flux immediately
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite

# Wait for CNPG to initialise
until kubectl get cluster authentik01-cnpg -n authentik01 \
  -o jsonpath='{.status.readyInstances}' | grep -q "1"; do sleep 5; done
echo "CNPG ready"

# Wait for Authentik server pod to become Ready (initial boot runs DB migrations)
kubectl rollout status deploy/authentik01-server -n authentik01 --timeout=10m
```

At this point Authentik has booted against a fresh database and applied its schema via
Django-style migrations. This is expected — the DB restore in Step 5 will replace it.

### Step 3 — Scale to zero before restore

```bash
kubectl scale deploy authentik01-server authentik01-worker -n authentik01 --replicas=0
```

### Step 4 — Dump from Docker host

```bash
# On the Docker host — find the postgres container name
docker ps --format '{{.Names}}' | grep -i postgres

# Dump the database
docker exec <postgres-container> \
  pg_dump -U authentik --clean --if-exists authentik > /tmp/authentik01.sql

# Copy to k3s machine
scp user@10.10.12.29:/tmp/authentik01.sql /tmp/authentik01.sql
```

### Step 5 — Wipe the fresh schema and restore

```bash
# Drop the schema created by Authentik's initial boot
kubectl exec -n authentik01 authentik01-cnpg-1 -- psql -U postgres authentik \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public; GRANT ALL ON SCHEMA public TO authentik;"

# Restore the dump
kubectl exec -i authentik01-cnpg-1 -n authentik01 -- psql -U postgres authentik \
  < /tmp/authentik01.sql
```

### Step 6 — Scale back up

```bash
kubectl scale deploy authentik01-server authentik01-worker -n authentik01 --replicas=1
kubectl rollout status deploy/authentik01-server -n authentik01 --timeout=5m
```

Check that Authentik recognised the existing database (not a fresh install):

```bash
kubectl logs -n authentik01 -l app.kubernetes.io/name=authentik01,app.kubernetes.io/component=server --tail=20
```

### Step 7 — Sync media and templates (if applicable)

Find where the NFS PVCs landed on TrueNAS:

```bash
kubectl get pv \
  $(kubectl get pvc authentik01-media -n authentik01 -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.nfs.path}'
```

If the Docker volumes have content, rsync from the Docker host to TrueNAS:

```bash
# Run on Docker host — single-quote the path to prevent ${} expansion
rsync -av /var/lib/docker/volumes/containers_vol/authentik/media/ \
  user@10.10.10.5:'/mnt/nvme2tb/k8s/pvs/<pv-path-for-authentik01-media>/'

rsync -av /var/lib/docker/volumes/containers_vol/authentik/custom-templates/ \
  user@10.10.10.5:'/mnt/nvme2tb/k8s/pvs/<pv-path-for-authentik01-templates>/'
```

### Step 8 — Cut over DNS and decommission Docker stack

Once the k3s instance is confirmed healthy:

1. Update DNS for `authentik01.local.safeqbit.com` to point to the k3s ingress IP
   (the MetalLB LoadBalancer IP of ingress-nginx), not `10.10.12.29`
2. Stop the Portainer stack on the Docker host
3. Optionally remove the Docker host VM if authentik01 was its only workload

---

## Key Files

```
apps/safeqbit-local-hq/authentik01/
├── 01-namespace.yaml          Namespace, backup.policy/velero: biweekly
├── 02-sealed-secrets.yaml     SealedSecret: AUTHENTIK_SECRET_KEY
├── 03-cnpg-cluster.yaml       CNPG Cluster: postgres 16, instances: 2, Longhorn 10Gi
├── 04-cnpg-scheduled-backup.yaml  Daily 02:00 UTC VolumeSnapshot
├── 06-shared-pvcs.yaml        media (5Gi) + templates (1Gi) on nfs-truenas (RWX)
├── 07-helmrelease.yaml        HelmRelease: authentik chart 2025.10.3, redis subchart on
└── 08-ingress.yaml            authentik01.local.safeqbit.com, letsencrypt-prod

infrastructure/safeqbit-local-hq/configs/
└── velero-schedule-authentik01.yaml   Biweekly 1st & 15th at 03:00 UTC, 6-month retention
```

---

## Useful Commands

```bash
# Check all authentik01 pods
kubectl get pods -n authentik01

# Tail server logs
kubectl logs -n authentik01 -l app.kubernetes.io/component=server -f

# Tail worker logs
kubectl logs -n authentik01 -l app.kubernetes.io/component=worker -f

# Connect to the CNPG database
kubectl exec -it -n authentik01 authentik01-cnpg-1 -- psql -U postgres authentik

# Check CNPG cluster health
kubectl get cluster -n authentik01 authentik01-cnpg

# Check Velero schedule
kubectl get schedule authentik01-biweekly -n velero

# Force Flux reconcile
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite
```
