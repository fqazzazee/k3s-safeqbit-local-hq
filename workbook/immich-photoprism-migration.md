# Immich & Photoprism Migration + nfs-bulk-media StorageClass

**Date:** 2026-05-15  
**Cluster:** safeqbit-local-hq  
**Namespaces:** `immich`, `photoprism`  
**Hostnames:** `immich.local.safeqbit.com`, `photoprism.local.safeqbit.com`

---

## What Changed and Why

Both apps were running as a single Portainer stack on a Docker host, sharing macvlan IPs on the LAN and mounting NFS paths directly from the host. Moving to k3s gives us:

- GitOps - the full stack is in version control
- Separate namespaces - Immich and Photoprism are independent apps that happen to share a photo library, not one monolithic stack
- Velero backups to B2 from per-namespace Schedules in `infrastructure/safeqbit-local-hq/configs/`: `immich-bimonthly` (1st + 16th, 03:30 UTC, 60d) and `photoprism-bimonthly` (13th + 28th, 03:00 UTC, 60d). They select by `includedNamespaces`; the `backup.policy/velero` namespace label is a leftover that nothing reads
- Ingress + cert-manager TLS instead of macvlan IPs
- Resource allocation in a dedicated file (`09-resources.yaml` / `06-resources.yaml`) with an optional node-pinning escape hatch for when hardware is upgraded

---

## New Infrastructure: nfs-bulk-media StorageClass

### What it is

A second NFS provisioner (`nfs-subdir-external-provisioner`) pointed at the TrueNAS Media share. It complements the existing `nfs-truenas` StorageClass (which provisions from the NVMe k8s PV pool) by providing dynamic RWX provisioning from the bulk spinning-disk Media share.

| StorageClass | Server | Base Path | Purpose |
|---|---|---|---|
| `nfs-truenas` | 10.10.10.5 | `/mnt/nvme2tb/k8s/pvs` | Fast NVMe-backed RWX storage for app data (Netbox media, etc.) |
| `nfs-bulk-media` | 10.10.10.5 | `/mnt/mach2/mach2nas/Media` | Bulk media share - large, slow, for photo libraries |

### How it is used

Immich and Photoprism do **not** use dynamic provisioning from `nfs-bulk-media`. Their photo library directories already exist on the TrueNAS Media share and contain live data, so moving them into subdir-provisioned paths would require copying TBs. Instead, they use **static PVs** (`storageClassName: ""`) that point directly to the existing NFS paths - zero data migration.

`nfs-bulk-media` exists for future workloads that need a new RWX directory on the bulk array. Requesting a PVC with `storageClassName: nfs-bulk-media` will dynamically create a new subdirectory under `/mnt/mach2/mach2nas/Media/`.

### File

```
infrastructure/safeqbit-local-hq/controllers/nfs-bulk.yaml
```

---

## Architecture: Old vs New

### Old Setup (Portainer stack `immich`)

```
Docker host: macvlan IPs 10.10.12.26 (immich), 10.10.12.27 (photoprism)
├── immich-server        (ghcr.io/immich-app/immich-server:release)
├── immich-machine-learning
├── redis (valkey:8)
├── postgres             (ghcr.io/immich-app/postgres:14-vectorchord...)
└── photoprism           (photoprism/photoprism:latest)

NFS mounts on Docker host (/mnt/nfs/Media/...):
  immich-server:   /data      → /mnt/nfs/Media/immich
                   /config    → local Docker volume
                   /external  → /mnt/nfs/Media/Memories/originals (ro)
                   /export    → /mnt/nfs/Media/Immich-Export
  ML:              /cache     → local Docker volume
  photoprism:      /originals → /mnt/nfs/Media/Memories/originals (rw)
                   /storage   → /mnt/nfs/Media/Memories/storage
                   /import    → /mnt/nfs/Media/Memories/import
```

### New Setup (k3s)

```
┌──────────────────────────────────────────────────────────────────┐
│                         immich namespace                         │
│                                                                  │
│  ┌──────────────────────┐   ┌────────────────────────────────┐  │
│  │ Deployment: postgres │   │  Deployment: immich-server     │  │
│  │ ghcr.io/immich-app/  │   │  wait-for-db initContainer     │  │
│  │ postgres:14-vector…  │   │  → /data  (NFS static PV)      │  │
│  │ strategy: Recreate   │   │  → /config (Longhorn 1Gi)      │  │
│  │                      │   │  → /external (NFS, readOnly)   │  │
│  │ ┌──────────────────┐ │   │  → /export  (NFS static PV)   │  │
│  │ │ Longhorn PVC 10Gi│ │   └────────────────────────────────┘  │
│  │ └──────────────────┘ │                                       │
│  └──────────────────────┘   ┌────────────────────────────────┐  │
│                             │  Deployment: machine-learning   │  │
│  ┌──────────────────────┐   │  → /cache (Longhorn 10Gi)      │  │
│  │ Deployment: redis    │   └────────────────────────────────┘  │
│  │ valkey:8, ephemeral  │                                       │
│  └──────────────────────┘   ┌────────────────────────────────┐  │
│                             │  Static NFS PVs                 │  │
│  ┌──────────────────────┐   │  immich-data-pv    → /immich   │  │
│  │ SealedSecret         │   │  immich-export-pv  → /Immich-  │  │
│  │ immich-credentials   │   │                      Export    │  │
│  │ • db-password        │   │  immich-external-pv → /Memories│  │
│  │ • db-username        │   │                      /originals│  │
│  │ • db-name            │   └────────────────────────────────┘  │
│  └──────────────────────┘                                       │
│                             ┌────────────────────────────────┐  │
│  ┌──────────────────────┐   │  Ingress                        │  │
│  │ 09-resources.yaml    │   │  immich.local.safeqbit.com      │  │
│  │ (kustomize patch)    │   │  proxy-body-size: 0             │  │
│  │ CPU/mem per workload │   │  proxy-timeout: 600s            │  │
│  │ + nodeSelector stub  │   └────────────────────────────────┘  │
│  └──────────────────────┘                                       │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                       photoprism namespace                       │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Deployment: photoprism                                   │   │
│  │  seccompProfile: Unconfined (libraw + ffmpeg syscalls)    │   │
│  │  → /originals (NFS static PV, RW - Photoprism sorts here)│   │
│  │  → /storage   (NFS static PV - SQLite DB lives here)     │   │
│  │  → /import    (NFS static PV - drop folder)              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────┐   ┌────────────────────────────┐   │
│  │ SealedSecret            │   │ Ingress                     │   │
│  │ photoprism-credentials  │   │ photoprism.local.safeqbit.  │   │
│  │ • admin-password        │   │ com, proxy-body-size: 0     │   │
│  └─────────────────────────┘   └────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions

### Why Photoprism and Immich are separate apps (not one namespace)

In Docker Compose they were a single stack for deployment convenience - one `docker compose down` stops everything. On k3s there is no benefit to coupling them. They have different update cadences, different resource profiles, and independent failure domains. Separating them means you can restart Photoprism without touching Immich and vice versa.

### Why Immich uses a plain Deployment for Postgres instead of CNPG

Immich requires `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` - a custom image that bundles the `vectorchord` and `pgvectors` extensions used for CLIP similarity search and face clustering. CNPG is incompatible with this image for two reasons:

1. The CNPG operator expects its own Postgres images (or images built on its base) with specific UID conventions (UID 26) and the `barman-cloud-*` tools for backup automation.
2. The Immich image is based on the upstream `postgres:14` which uses UID 999 and has no barman tooling.

The plain Deployment approach matches the original Docker Compose setup and has no practical disadvantage for a single-instance personal deployment. Velero backs up the Longhorn PVC (`immich-postgres-data`, 10Gi).

To connect manually:
```bash
PGPASS=$(kubectl get secret immich-credentials -n immich \
  -o jsonpath='{.data.db-password}' | base64 -d)
kubectl exec -it -n immich deploy/immich-postgres -- \
  env PGPASSWORD="$PGPASS" psql -U postgres-user immich
```

### Why Immich Redis has no password and no persistence

Immich's Redis holds ephemeral state: job queue entries, active WebSocket sessions, thumbnail generation tasks. On restart, the server reconnects and rebuilds the queue from the database. The original Compose stack also had no Redis password. Adding one would require `REDIS_PASSWORD` in the server env and the `requirepass` arg in the Valkey container - unnecessary complexity for an internal cluster service with no external exposure.

### Why Photoprism needs `seccompProfile: Unconfined`

Photoprism runs `libraw` (for RAW photo processing) and `ffmpeg` (for video thumbnails) as in-process libraries. Both call syscalls that are blocked by the default Kubernetes seccomp profile (`RuntimeDefault`). The original Compose stack set `seccomp:unconfined` for the same reason. Setting it at pod level applies to all containers in the pod.

### Shared NFS path: Memories/originals

Both Immich and Photoprism mount `/mnt/mach2/mach2nas/Media/Memories/originals`:

- **Photoprism** mounts it read-write - it organises and imports photos here
- **Immich** mounts it read-only at `/external` - it reads the same library for its timeline view

In Kubernetes, a single PV can only bind to one PVC. Two separate static PVs point to the same NFS path - one in each namespace. The NFS server (TrueNAS) handles concurrent multi-client access.

| PV | Namespace | Mount | Access |
|---|---|---|---|
| `immich-external-pv` | immich | `/external` | ReadOnly |
| `photoprism-originals-pv` | photoprism | `/photoprism/originals` | ReadWrite |

### NFS PV capacity values are informational

For NFS PVs the `capacity.storage` field has no enforcement - the actual quota is managed by TrueNAS, not Kubernetes. The values set here (6Ti for originals, 1Ti for import, 500Gi for storage/export) exist solely so that `kubectl get pv` shows meaningful output. They will not cause scheduling failures or prevent writes once the TrueNAS quota is reached.

### Resource allocation as a kustomize patch

All `resources:` blocks are removed from the deployment manifests and live exclusively in:

- `immich/09-resources.yaml` - postgres, redis, server (inc. init container), ML
- `photoprism/06-resources.yaml` - photoprism

These files are strategic merge patches applied via `patchesStrategicMerge` in each `kustomization.yaml`. Each deployment section includes a commented-out `nodeSelector` block - uncomment and set `kubernetes.io/hostname` to pin that workload to a specific node when hardware is upgraded.

```bash
# Find node hostnames
kubectl get nodes
```

---

## NFS Share Map

Server: `10.10.10.5`  
Base: `/mnt/mach2/mach2nas/Media`

| NFS path | PV name | Namespace | Container mount | Access | Capacity |
|---|---|---|---|---|---|
| `/immich` | `immich-data-pv` | immich | `/data` | RW | 1Ti |
| `/Immich-Export` | `immich-export-pv` | immich | `/export` | RW | 500Gi |
| `/Memories/originals` | `immich-external-pv` | immich | `/external` | RO | 6Ti |
| `/Memories/originals` | `photoprism-originals-pv` | photoprism | `/photoprism/originals` | RW | 6Ti |
| `/Memories/storage` | `photoprism-storage-pv` | photoprism | `/photoprism/storage` | RW | 500Gi |
| `/Memories/import` | `photoprism-import-pv` | photoprism | `/photoprism/import` | RW | 1Ti |

---

## Migration Runbook

### What actually happened

Both apps were migrated with zero manual data movement.

**Photoprism** required no DB migration - its SQLite database lives inside `/Memories/storage` on TrueNAS, which k3s already mounts via the static NFS PV. The pod came up and immediately had access to the full library.

**Immich** was restored from Immich's own built-in backup (Administration → Jobs → Backup). The NFS photo library (`/immich`, `/Memories/originals`) was already accessible via the static NFS PVs - no file copying needed.

### Sealing the secrets

Credentials were sealed directly from the raw Secret manifests using the `kubeseal` binary downloaded to `/tmp`:

```bash
# Download kubeseal matching the controller version
curl -sL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.3/kubeseal-0.27.3-linux-amd64.tar.gz" \
  | tar -xz -C /tmp kubeseal

# Seal immich
cd apps/safeqbit-local-hq/immich
kubeseal --controller-namespace kube-system --format yaml \
  < 02-sealed-secrets.yaml > /tmp/s.yaml && mv /tmp/s.yaml 02-sealed-secrets.yaml

# Seal photoprism
cd ../photoprism
kubeseal --controller-namespace kube-system --format yaml \
  < 02-sealed-secrets.yaml > /tmp/s.yaml && mv /tmp/s.yaml 02-sealed-secrets.yaml
```

### Deploying

```bash
# Push manifests
git push

# Force Flux to pick up immediately (avoids waiting up to 10m for the next poll)
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite

# Watch rollout
kubectl get pods -n immich -w
kubectl get pods -n photoprism -w
```

---

## Issues Found During Deployment

### 1. `apps` kustomization blocked by invalid CNPG cron expression

**Symptom:** All kustomizations showed `READY: False` with:
```
ScheduledBackup/affine/affine-cnpg-backup dry-run failed (Invalid):
spec.schedule: Invalid value: "0 3 * * 0":
Beginning of range (0) below minimum (1)
```

**Cause:** The CNPG admission webhook uses a cron parser that requires day-of-week values to be `1-7` (Monday = 1, Sunday = 7). Standard POSIX cron uses `0` for Sunday. The AFFiNE weekly backup schedule had `0 3 * * 0`.

**Fix:** Change `* * 0` to `* * 7` in `apps/safeqbit-local-hq/affine/04-cnpg-scheduled-backup.yaml`.

**Rule:** For any CNPG `ScheduledBackup`, never use `0` for Sunday - always use `7`.

### 2. Immich Postgres CrashLoopBackOff on first boot

**Symptom:**
```
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory, perhaps due to it being a mount point.
Using a mount point directly as the data directory is not recommended.
Create a subdirectory under the mount point.
```

**Cause:** Longhorn formats the PVC filesystem with a `lost+found` directory at the root. Postgres's `initdb` refuses to initialise into a non-empty directory.

**Fix:** Add `PGDATA=/var/lib/postgresql/data/pgdata` to the postgres container env. Postgres then initialises into the subdirectory, which is empty.

```yaml
env:
  - name: PGDATA
    value: /var/lib/postgresql/data/pgdata
```

**Rule:** Every Postgres Deployment (not CNPG - CNPG handles this internally) on a Longhorn PVC needs `PGDATA` set to a subdirectory. See also the Vaultwarden and Netbox CNPG notes for why CNPG is preferred when the image allows it.

---

## Upgrading Immich

**Current:** server + machine-learning `v3.3.6` (since 2026-11). Postgres
`ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`, unchanged
since the migration.

Rules:

- Pin an exact tag. Never use the floating `:release` tag (it jumps majors on
  a pod reschedule).
- Bump server (`06-immich-server.yaml`) and ML (`07-immich-ml.yaml`) in the
  same commit, to the same tag.
- On every bump, compare the postgres image in upstream
  `docker/docker-compose.yml` at the new tag with `03-postgres.yaml`. If it
  changed, the vectorchord/pgvectors extension upgrade needs its own plan and
  restore point; do not fold it into a server bump.
- Read every release note in the range for schema migrations and env-var
  renames. The server runs migrations before it opens port 2283; the
  startupProbe gives it 10 minutes. Immich has no supported downgrade once a
  migration has run.

Restore point before any bump with migrations (the July 2026 pattern):

```bash
# 1. Longhorn snapshot of the DB volume (class longhorn-velero)
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: immich-postgres-preupgrade-$(date +%Y%m%d)
  namespace: immich
spec:
  volumeSnapshotClassName: longhorn-velero
  source:
    persistentVolumeClaimName: immich-postgres-data
EOF
kubectl get volumesnapshot -n immich -w   # wait for READYTOUSE=true

# 2. Logical dump to the workstation
kubectl exec -n immich deploy/immich-postgres -- \
  sh -c 'pg_dump -U "$POSTGRES_USER" --no-owner --no-acl "$POSTGRES_DB"' \
  | gzip > immich-preupgrade-$(date +%Y%m%d).sql.gz
```

Delete the VolumeSnapshot once the new version has soaked (it holds a Longhorn
snapshot of the whole 10Gi volume).

### Version history

| Date | Version | Notes |
|---|---|---|
| 2026-07-19 | v3.0.3 | Major; mobile apps must be v3+, legacy timeline removed |
| 2026-09-22 | v3.2.2 | |
| 2026-11 | v3.3.6 | 5 schema migrations (people sharing, asset-face state, OAuth id cleanup, GeoNames country rename). 3.3.5 = Node Oct-2026 security release + certifi (eMudhra distrust); 3.3.6 turns off onnxruntime telemetry in the ML container. Postgres image unchanged |

---

## Key Files

```
infrastructure/safeqbit-local-hq/controllers/
└── nfs-bulk.yaml                  nfs-bulk-media StorageClass (HelmRelease)

apps/safeqbit-local-hq/immich/
├── 01-namespace.yaml              Namespace (its backup.policy label is unused)
├── 02-sealed-secrets.yaml         SealedSecret: db-password, db-username, db-name
├── 03-postgres.yaml               Deployment (plain, strategy: Recreate) + Service
├── 04-redis.yaml                  Valkey Deployment + Service (no auth, no PVC)
├── 05-pvcs.yaml                   3× Longhorn PVCs + 3× static NFS PV/PVC pairs
├── 06-immich-server.yaml          Deployment (wait-for-db init) + Service
├── 07-immich-ml.yaml              Deployment + Service
├── 08-ingress.yaml                immich.local.safeqbit.com, proxy-body-size: 0
├── 09-resources.yaml              kustomize patch: CPU/mem + nodeSelector stubs
└── kustomization.yaml

apps/safeqbit-local-hq/photoprism/
├── 01-namespace.yaml              Namespace (its backup.policy label is unused)
├── 02-sealed-secrets.yaml         SealedSecret: admin-password
├── 03-pvcs.yaml                   3× static NFS PV/PVC pairs
├── 04-photoprism.yaml             Deployment (seccompProfile: Unconfined) + Service
├── 05-ingress.yaml                photoprism.local.safeqbit.com, proxy-body-size: 0
├── 06-resources.yaml              kustomize patch: CPU/mem + nodeSelector stub
└── kustomization.yaml
```

---

## Useful Commands

```bash
# ── Immich ────────────────────────────────────────────────────────────────

# All pods
kubectl get pods -n immich

# Server logs (API requests, job output)
kubectl logs -n immich deploy/immich-server -f

# ML logs (model loading, inference)
kubectl logs -n immich deploy/immich-machine-learning -f

# Postgres logs
kubectl logs -n immich deploy/immich-postgres -f

# Connect to Immich DB
PGPASS=$(kubectl get secret immich-credentials -n immich \
  -o jsonpath='{.data.db-password}' | base64 -d)
kubectl exec -it -n immich deploy/immich-postgres -- \
  env PGPASSWORD="$PGPASS" psql -U postgres-user immich

# Manual pg_dump to NFS
kubectl exec -n immich deploy/immich-postgres -- \
  env PGPASSWORD="$PGPASS" \
  pg_dump -U postgres-user --no-owner --no-acl immich \
  > /mnt/nfs/Media/immich-db-$(date +%F).sql

# Scale server down/up (e.g. before a DB restore)
kubectl scale deploy immich-server -n immich --replicas=0
kubectl scale deploy immich-server -n immich --replicas=1

# ── Photoprism ────────────────────────────────────────────────────────────

# All pods
kubectl get pods -n photoprism

# Logs
kubectl logs -n photoprism deploy/photoprism -f

# ── Both ─────────────────────────────────────────────────────────────────

# Check all NFS PVs
kubectl get pv | grep -E "immich|photoprism"

# Check ingress IPs
kubectl get ingress -n immich -n photoprism

# Pin a workload to a specific node (edit 09-resources.yaml or 06-resources.yaml,
# uncomment the nodeSelector block, set the hostname, commit and push)
kubectl get nodes   # → find the hostname to use
```

---

## Lessons Learned

**CNPG rejects `0` for Sunday in cron expressions.** CNPG's webhook validates day-of-week as `1-7` only. Any `ScheduledBackup` using `* * 0` for Sunday will fail admission and block the entire apps kustomization from reconciling. Use `* * 7`.

**Every Postgres Deployment on a Longhorn PVC needs `PGDATA` set to a subdirectory.** Longhorn creates a `lost+found` at the PVC root. Postgres `initdb` refuses a non-empty directory. Set `PGDATA=/var/lib/postgresql/data/pgdata`. CNPG handles this internally - it only affects plain Deployment-based Postgres like Immich's custom image.

**Photoprism needs no DB migration when the storage path is NFS-backed.** The SQLite database lives inside the storage volume. As long as the NFS PVC points to the same existing directory, the pod boots with full history intact. No dump/restore needed.

**Two static PVs can point to the same NFS path.** Kubernetes enforces 1 PV → 1 PVC binding, but two separate PVs pointing to the same NFS export is valid. Use `claimRef` in each PV to prevent accidental cross-binding.

**`proxy-body-size: 0` is required for photo/video upload apps.** nginx's default is 1m. Immich and Photoprism both upload multi-MB to multi-GB files. Without this annotation the ingress silently rejects large uploads with a 413 error. Always set it on media-heavy ingresses.

---

## Upgrading PhotoPrism

**Current:** `photoprism/photoprism:261029` (since 2026-11). Tags are dates
(YYMMDD); pin the exact tag and never `:latest`.

Things that matter on every bump:

- **The index is SQLite on NFS** (`/photoprism/storage/index.db`, on the
  `photoprism-storage` static PV). Velero skips NFS, so the only copies are
  TrueNAS snapshots of `Memories/storage` and PhotoPrism's own daily SQL dumps
  in `/photoprism/storage/backup/sqlite3/`. Those keep only 3 days
  (`PHOTOPRISM_BACKUP_RETAIN` default), so a pre-upgrade dump rotates out three
  days after the upgrade. Copy the DB aside first (below).
- **`strategy: Recreate`** (since 261029). The volume is RWX, so a
  RollingUpdate would happily run two pods on one `index.db`, with the new one
  migrating the schema while the old one writes.
- **startupProbe, 10 minutes.** Schema migrations run before port 2342 opens.
- **`PHOTOPRISM_DISABLE_MCP=true`.** 261029 enabled the MCP endpoint (AI
  agents searching the library) by default. Nothing here uses it.
  `PHOTOPRISM_OAUTH_APPS` stays at its default `registered` (only apps an admin
  registers can use the new OAuth sign-in flow).

Copy the DB aside before a bump with schema changes. This uses SQLite's online
backup API: the source is opened read-only and only the new file is written,
so PhotoPrism can keep running.

```bash
D=$(date +%Y%m%d)
kubectl exec -n photoprism deploy/photoprism -- \
  sqlite3 -readonly /photoprism/storage/index.db \
  ".backup '/photoprism/storage/index.db.preupgrade-$D'"
kubectl exec -n photoprism deploy/photoprism -- \
  sqlite3 -readonly /photoprism/storage/index.db.preupgrade-$D 'PRAGMA integrity_check;'   # ok
# Optional off-cluster copy
kubectl exec -n photoprism deploy/photoprism -- \
  cat /photoprism/storage/index.db.preupgrade-$D > photoprism-index-preupgrade-$D.db
```

Rollback: revert the image tag in Git, suspend the `apps` Kustomization and
scale PhotoPrism to 0, copy the saved file over `index.db` (the share is also
mounted on TrueNAS, or use a throwaway pod on the `photoprism-storage` PVC),
then scale up and resume. Delete the copy once the new version has soaked.

### Version history

| Date | Version | Notes |
|---|---|---|
| 2026-10-23 | 261007 | |
| 2026-11 | 261029 | DB layer upgrade (GORM v1.9 -> v1.31) with schema migrations; MCP on by default (disabled here); files with `\|"?*%<>{}:~$` in their names are no longer indexed. libheif (23 GHSA), Go 1.27.3 (11 CVE), ONNX Runtime 1.31 |
