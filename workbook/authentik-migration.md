# Authentik: Full Migration History

**Cluster:** safeqbit-local-hq

| Phase | Date | What happened |
|---|---|---|
| 1 | 2026-05-14 | Hand-rolled StatefulSet Postgres → CloudNativePG |
| 2 | 2026-05-15 | Prod Authentik on Docker host (10.10.12.29) → k3s; old namespace decommissioned |

---

## Current State

```
Namespace: authentik
├── CNPG Cluster: authentik-cnpg (Postgres 16, instances: 2, Longhorn 10Gi)
│   ├── authentik-cnpg-1  (primary)
│   └── authentik-cnpg-2  (hot standby)
├── HelmRelease: authentik (chart: authentik 2025.10.3, redis subchart enabled)
├── NFS PVCs: authentik-media (5Gi), authentik-templates (1Gi) — nfs-truenas RWX
└── Ingress: authentik01.local.safeqbit.com → authentik-server:80 (letsencrypt-prod)
```

---

## Phase 1 — StatefulSet Postgres → CloudNativePG (2026-05-14)

### What changed and why

Authentik's database was a hand-rolled Postgres 16 StatefulSet — a single pod mounting a
Longhorn PVC. It worked but had three problems:

1. **Crash-inconsistent backups.** Velero snapshotted the PVC while Postgres was running.
   Postgres keeps dirty pages in `shared_buffers` that may not have been flushed to disk.
   A restore from such a snapshot could require crash recovery and might be logically
   inconsistent.
2. **No failover.** If the pod died Kubernetes restarted it, but in-flight transactions
   were lost and there was no hot standby.
3. **Manual credential management.** `POSTGRES_PASSWORD` lived in the SealedSecret and had
   to be kept in sync manually.

CloudNativePG (CNPG) solves all three.

### How CNPG works

The CNPG operator runs in `cnpg-system` and watches all namespaces for `Cluster` CRs.
When you create one, the operator:

1. Creates one Postgres pod per `spec.instances`
2. Creates a Longhorn PVC for each pod (`<cluster-name>-<n>`)
3. Runs `initdb` with the configured database and owner
4. Creates three Services kept up to date as primaries change:
   - `authentik-cnpg-rw` — current primary (all writes go here)
   - `authentik-cnpg-ro` — hot standbys only
   - `authentik-cnpg-r` — any instance
5. Auto-generates `authentik-cnpg-app` secret with 11 keys (`username`, `password`,
   `host`, `uri`, `jdbc-uri`, etc.)

The HelmRelease reads `AUTHENTIK_POSTGRESQL__PASSWORD` directly from `authentik-cnpg-app`,
so no manual password management is needed.

### How CNPG backups work

**Before (Velero DB-unaware):**
```
Velero fires → tells Longhorn to snapshot the PVC → snapshot taken while Postgres runs
→ dirty pages may still be in RAM → restore requires crash recovery
```

**Now (CNPG DB-aware):**
```
ScheduledBackup fires (02:00 UTC daily)
  → CNPG issues CHECKPOINT to Postgres (flush all dirty pages to disk)
  → Postgres confirms checkpoint complete
  → CNPG triggers VolumeSnapshot via Longhorn CSI
  → Snapshot represents a guaranteed-clean, consistent state
  → Backup object created; owns the VolumeSnapshot (GC'd together)
```

Velero continues to run biweekly and covers the rest of the namespace (media PVC,
templates PVC, sealed secret). Use the CNPG ScheduledBackup for database-specific restores.

### Phase 1 runbook (what was done)

1. Deployed CNPG Cluster CR (`authentik-cnpg`, instances: 1 at the time)
2. Suspended the HelmRelease before the push — **critical lesson below**
3. Waited for CNPG to initialise (`authentik-cnpg-app` secret created)
4. Wiped the empty schema CNPG created on first boot
5. Streamed `pg_dump` from the old StatefulSet pod directly into CNPG via a single piped
   `kubectl exec` command (no intermediate file)
6. Verified row counts matched between old and new
7. Resumed the HelmRelease; Authentik pointed at `authentik-cnpg-rw`
8. Deleted the old StatefulSet, Service, and PVC manually (prune: false)
9. Re-sealed `authentik-credentials` with only `AUTHENTIK_SECRET_KEY` (dropped the now-
   unused `POSTGRES_PASSWORD` and `AUTHENTIK_POSTGRESQL__PASSWORD` keys)

---

## Phase 2 — Docker Host → k3s (2026-05-15)

### What changed and why

The production Authentik instance (`authentik01.local.safeqbit.com`) was running as a
Portainer stack on a standalone Docker host with a macvlan IP (`10.10.12.29`):

```
Docker host: 10.10.12.29
├── postgres:16-alpine    ← hand-rolled, password in stack.env
├── authentik-server      ← ghcr.io/goauthentik/server:2025.10.3
└── authentik-worker      ← ghcr.io/goauthentik/server:2025.10.3

Media:     /var/lib/docker/volumes/containers_vol/authentik/media
Templates: /var/lib/docker/volumes/containers_vol/authentik/custom-templates
```

Problems: separate VM to maintain, no GitOps, no application-consistent backups, no HA.

The k3s migration landed everything under the existing `authentik` namespace pattern
(the Phase 1 namespace was decommissioned once the Docker host data was live).

### Phase 2 runbook (what was done)

**Step 1 — Sealed the credential**

Retrieved `AUTHENTIK_SECRET_KEY` from `stack.env` on the Docker host and sealed it
for the target namespace:

```bash
kubectl create secret generic authentik-credentials \
  --namespace authentik \
  --dry-run=client \
  --from-literal=AUTHENTIK_SECRET_KEY='<value>' \
  -o yaml | \
/tmp/kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  -o yaml > apps/safeqbit-local-hq/authentik/02-sealed-secrets.yaml
```

`AUTHENTIK_SECRET_KEY` must match the Docker stack exactly — Authentik uses it to sign
sessions and tokens. Changing it invalidates all active sessions.

**Step 2 — Pushed manifests and waited for initial boot**

```bash
git add apps/safeqbit-local-hq/authentik/ \
        infrastructure/safeqbit-local-hq/configs/velero-schedule-authentik.yaml \
        infrastructure/safeqbit-local-hq/configs/kustomization.yaml \
        apps/safeqbit-local-hq/kustomization.yaml
git commit -m "apps: add Authentik migration to k3s"
git push origin main

kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite

until kubectl get cluster authentik-cnpg -n authentik \
  -o jsonpath='{.status.readyInstances}' | grep -q "1"; do sleep 5; done

kubectl rollout status deploy/authentik-server -n authentik --timeout=10m
```

At this point Authentik booted against a fresh database (ran schema migrations). This is
expected — the restore in Step 4 overwrites it.

**Step 3 — Scaled to zero**

```bash
kubectl scale deploy authentik-server authentik-worker -n authentik --replicas=0
```

Always scale to zero before a restore. Active writes during a restore corrupt the result.

**Step 4 — Dumped from Docker host and restored**

```bash
# On Docker host
docker exec authentik-postgres \
  pg_dump -U authentik --clean --if-exists authentik > /tmp/authentik.sql
scp user@10.10.12.29:/tmp/authentik.sql /tmp/authentik.sql

# On k3s machine — wipe the fresh schema first
kubectl exec -n authentik authentik-cnpg-1 -- psql -U postgres authentik \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public; GRANT ALL ON SCHEMA public TO authentik;"

# Restore
kubectl exec -i authentik-cnpg-1 -n authentik -- psql -U postgres authentik \
  < /tmp/authentik.sql
```

`--clean --if-exists` adds `DROP ... IF EXISTS` before every `CREATE`, making the restore
idempotent. Dropping the whole public schema first handles circular dependencies that
`--clean` alone can't resolve on a Django-migrated database.

**Step 5 — Scaled back up and cut over DNS**

```bash
kubectl scale deploy authentik-server authentik-worker -n authentik --replicas=1
kubectl rollout status deploy/authentik-server -n authentik --timeout=5m
```

Updated DNS for `authentik01.local.safeqbit.com` from `10.10.12.29` → `10.10.13.50`
(k3s ingress MetalLB IP). Stopped the Portainer stack on the Docker host.

---

## Key Files

```
apps/safeqbit-local-hq/authentik/
├── 01-namespace.yaml             Namespace; backup.policy/velero: biweekly
├── 02-sealed-secrets.yaml        SealedSecret: AUTHENTIK_SECRET_KEY only
├── 03-cnpg-cluster.yaml          CNPG Cluster: postgres 16, instances: 2, Longhorn 10Gi
├── 04-cnpg-scheduled-backup.yaml Daily 02:00 UTC VolumeSnapshot
├── 06-shared-pvcs.yaml           media (5Gi) + templates (1Gi) on nfs-truenas (RWX)
├── 07-helmrelease.yaml           authentik chart 2025.10.3; redis subchart on
└── 08-ingress.yaml               authentik01.local.safeqbit.com; letsencrypt-prod

infrastructure/safeqbit-local-hq/configs/
└── velero-schedule-authentik.yaml   Biweekly 1st & 15th at 03:00 UTC; 6-month retention
```

---

## Useful Commands

```bash
# Pod status
kubectl get pods -n authentik

# Tail server / worker logs
kubectl logs -n authentik -l app.kubernetes.io/component=server -f
kubectl logs -n authentik -l app.kubernetes.io/component=worker -f

# CNPG cluster health
kubectl get cluster -n authentik authentik-cnpg

# Connect to the database
kubectl exec -it -n authentik authentik-cnpg-1 -- psql -U postgres authentik

# Check Velero schedule
kubectl get schedule authentik-biweekly -n velero

# Force Flux reconcile
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite
```

---

## Lessons Learned

**Suspend the HelmRelease before pushing DB-breaking changes.** Flux reconciles within
seconds of a push. If the new HelmRelease points at a fresh database, Authentik will
immediately run schema migrations and seed default data — contaminating the restore target.
The safe pattern is always:

```
suspend HelmRelease → push → wait for CNPG → dump/restore → resume HelmRelease
```

**`prune: false` means manual cleanup.** The apps Kustomization has `prune: false` — Flux
never deletes cluster resources when their manifests are removed from Git. Old StatefulSets,
namespaces, and Velero schedules must be deleted manually after removing them from Git.

**Sealed secrets are namespace-scoped.** A SealedSecret encrypted for namespace `foo`
cannot be decrypted in namespace `bar`. When renaming a namespace, always re-run `kubeseal`
— a `sed` rename of the metadata field leaves the ciphertext scoped to the old namespace
and the controller will silently fail to decrypt it.

**Scale to zero before any database restore.** If the application is writing new sessions
or state while you restore, the result is corrupted. Scale web and worker to 0 first,
restore, then scale back up.

**`kubectl exec -i ... < file` restore throughput is ~20–30 MB/s.** A 113MB dump takes
4–5 minutes through the kubectl API pipe. Check completion via `pg_stat_activity` (no
active restore connection) or `SELECT count(*) FROM pg_tables` (187 tables = full restore).

**`--clean --if-exists` + schema wipe is the reliable restore pattern.** `--clean` alone
can't resolve circular dependencies in a Django schema on top of an existing schema. Dropping
`public` entirely first gives a clean slate, then the `--clean` flags handle any
post-`initdb` CNPG objects that remain.
