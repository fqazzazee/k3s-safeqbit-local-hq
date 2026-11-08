# CNPG Strategy, Inventory & Restore

**Cluster:** safeqbit-local-hq
**Last updated:** 2026-11-08 (full rewrite: inventory, WAL sizing, restore procedure)

CloudNativePG runs every Postgres database in the cluster except Immich's. This
page covers what exists, why it is sized the way it is, how it is backed up, and
how to get a database back. [backup-strategy.md](backup-strategy.md) owns the
cluster-wide backup picture; this page owns the Postgres-specific parts of it.

---

## The operator

- **HelmRelease:** `infrastructure/safeqbit-local-hq/controllers/cnpg.yaml`,
  chart `cloudnative-pg` 0.29.1, operator **1.30.1**, namespace `cnpg-system`.
- **2 replicas** (`replicaCount: 2`) with its own PDB, so an operator pod on a
  drained node doesn't stop reconciliation.
- Watches every namespace. Each app declares its own `Cluster` in its own
  namespace, and the operator creates the rest:

| Created per cluster | What it is |
|---|---|
| `<cluster>-rw` Service | The current primary. Every app here connects to this |
| `<cluster>-ro` / `<cluster>-r` Services | Standbys only / any instance. Nothing here uses them |
| `<cluster>-app` Secret | `kubernetes.io/basic-auth` for the app user. Keys: `username`, `password`, `dbname`, `host`, `port`, `uri`, `jdbc-uri`, `fqdn-uri`, `fqdn-jdbc-uri`, `pgpass`, `user`. `host` is `<cluster>-rw` and `uri` is a ready-made DSN against it |
| `<cluster>-ca`, `-server`, `-replication` Secrets | Internal TLS and replication. Owned by the Cluster, so they go when it goes |
| `<cluster>-primary` PDB | `minAvailable: 1` on the primary. See [Drains](#drains-and-the-primary-pdb) |

Superuser access over the network is off (`enableSuperuserAccess` unset), but
`kubectl exec` into an instance and plain `psql` works: the pod lets the local
`postgres` user in over the socket.

There is no `kubectl cnpg` plugin installed. Everything below is plain
`kubectl`.

---

## Inventory

All five run `ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie`, on
`storageClass: longhorn` (3 Longhorn replicas per volume), with
`affinity.topologyKey: kubernetes.io/hostname` and `monitoring.enablePodMonitor: true`.

| Cluster | Namespace | Instances | Database / owner | Storage | Manifest |
|---|---|---|---|---|---|
| `affine-cnpg` | `affine` | 2 | `affine` / `affine` | 5Gi | `apps/safeqbit-local-hq/affine/03-cnpg-cluster.yaml` |
| `authentik-cnpg` | `authentik` | 2 | `authentik` / `authentik` | 10Gi | `apps/safeqbit-local-hq/authentik/03-cnpg-cluster.yaml` |
| `grafana-cnpg` | `monitoring` | 2 | `grafana` / `grafana` | 5Gi | `infrastructure/safeqbit-local-hq/configs/grafana-cnpg.yaml` |
| `netbox-cnpg` | `netbox` | 2 | `netbox` / `netbox` | 5Gi | `apps/safeqbit-local-hq/netbox/03-cnpg-cluster.yaml` |
| `pangolin-cnpg` | `pangolin` | 1 | `pangolin` / `pangolin` | 5Gi | `apps/safeqbit-local-hq/pangolin/02-cnpg-cluster.yaml` |

Data is small. On 2026-11-08 the `base/` directories were 38–68MB, except
authentik at 267MB. Most of each volume is the WAL ring (below).

**Instance numbers are not stable.** On 2026-11-08 the primaries were
`affine-cnpg-1`, `authentik-cnpg-2` (there is no `authentik-cnpg-1` any more),
`grafana-cnpg-1`, `netbox-cnpg-2` (`netbox-cnpg-1` is the replica) and
`pangolin-cnpg-1`. Switchovers and rebuilt replicas move these around, so never
write `<cluster>-1` into a runbook. Look the primary up instead:
[Find the current primary](#find-the-current-primary).

Not on CNPG:

| Workload | Database | Notes |
|---|---|---|
| Immich | Own `immich-postgres` Deployment (`ghcr.io/immich-app/postgres:14-vectorchord…`) | Needs Immich's custom image with vector extensions. Covered by Velero only |
| Vaultwarden, passzilla, uptime-kuma, home-assistant, qbit-labs | SQLite on a Longhorn RWO volume | Single replica each. See [vaultwarden.md](vaultwarden.md), [passzilla.md](passzilla.md) |
| PhotoPrism | SQLite on the static NFS PV | TrueNAS is its only protection |

Guacamole's `guacamole-cnpg` was deleted with the app on 2026-08-29. Its
manifests are still in `apps/safeqbit-local-hq/guacamole/` but the directory is
commented out of the apps kustomization.

### Postgres version pin

`imageName` is set explicitly on every cluster. The CNPG webhook stamps the
operator's default image into the live spec when a cluster is created and never
changes it afterwards, so operator upgrades left four clusters sitting on 18.3
until the pin went in (October 2026 wave). To move Postgres, change `imageName`
in all five manifests. A minor bump (18.x → 18.y) is a rolling restart: replicas
first, then a switchover (authentik, `primaryUpdateMethod: switchover`) or an
in-place primary restart (the rest). A major bump is a different procedure.
Don't do it by editing the tag.

---

## Instance count rationale

**2 instances** for affine, authentik, grafana and netbox. A standby turns a
node loss into a failover measured in seconds. With one instance, the pod has to
reschedule and reattach its volume, and the app is down for that whole window.
The cost is a second PVC: another 5Gi (10Gi for authentik), times 3 Longhorn
replicas.

- **authentik** is the one that really needs it. It fronts SSO for Grafana,
  Prometheus and Alertmanager. It also sets `primaryUpdateMethod: switchover`.
  Its server pods crash on dead DB connections, and crash-backoff turned a ~10s
  primary restart into ~5 minutes of SSO 503s during the CNPG 1.30 operator
  upgrade (2026-07-19). A switchover keeps a writable primary up throughout.
- **grafana** goes completely down without its DB.
- **affine, netbox**: user-facing, and the standby is cheap at these sizes.

**1 instance** for pangolin. It is an internal remote-access tool, and the
manifest's own note says to bump it to 2 if it becomes a critical auth path.
The single instance has one operational cost, covered under
[Drains](#drains-and-the-primary-pdb).

**3 instances** is not used. Promotion doesn't need quorum in CNPG, so a third
copy on a 3-node cluster buys very little.

---

## WAL sizing

Every cluster sets the same three parameters:

```yaml
postgresql:
  parameters:
    max_wal_size: "256MB"
    min_wal_size: "80MB"
    wal_keep_size: "128MB"
```

CNPG's defaults (`wal_keep_size` 512MB, `max_wal_size` 1GB) held `pg_wal` at
about 561MB on every cluster, whatever the database size. Velero snapshots the
whole PVC, so roughly 90% of every CNPG backup shipped to B2 was empty WAL ring.
The Layer-2 snapshots paid for it too: busy writers rewrite most of the ring
every day. These values went in on 2026-08-17 (PR #90) and took total pgdata
from 3.85GB to 1.74GB.

- **Both are `sighup` parameters.** CNPG applies them with a config reload, with
  no restart and no failover.
- **`pg_wal` does not shrink by itself.** Postgres only recycles segments at a
  checkpoint, and these databases are quiet enough that timed checkpoints get
  skipped. After changing the values, run `CHECKPOINT;` twice on the primary.
- **The floor is about 209MB, not the ~150MB the numbers suggest** (128MB kept,
  plus headroom and in-flight segments, ≈ 13 × 16MB). On 2026-11-08 every
  primary sat at 193–209MB.
- Safe because CNPG's HA replication slots hold WAL for the standbys anyway.
  `wal_keep_size` is only a backstop for a standby that has lost its slot.

Check one:

```bash
kubectl -n <ns> exec <primary-pod> -c postgres -- \
  du -sh /var/lib/postgresql/data/pgdata/pg_wal /var/lib/postgresql/data/pgdata/base
```

---

## Backups

Two layers protect each database. Neither is a WAL archive, so **there is no
point-in-time recovery**. Every restore lands on the exact moment a snapshot was
taken.

| Cluster | Layer 2: CNPG ScheduledBackup (in-cluster) | Kept | Layer 3: Velero → B2 (off-site) | TTL |
|---|---|---|---|---|
| `affine-cnpg` | `affine-cnpg-backup` `0 0 2 * * *`, daily 02:00 | 7 | `affine-biweekly` 2nd + 17th 03:00 | 21d |
| `netbox-cnpg` | `netbox-cnpg-backup` `0 15 2 * * *`, daily 02:15 | 7 | `netbox-biweekly` 4th + 19th 03:00 | 21d |
| `grafana-cnpg` | `grafana-cnpg-backup` `0 30 2 * * *`, daily 02:30 | 7 | `monitoring-bimonthly` 9th + 24th 03:00 | 21d |
| `authentik-cnpg` | `authentik-cnpg-backup` `0 45 2 * * 0`, Sun 02:45 | 6 | `authentik-bimonthly` 5th + 20th 03:00 | 21d |
| `pangolin-cnpg` | `pangolin-cnpg-backup` `0 30 3 * * 0`, Sun 03:30 | 4 | `pangolin-monthly` 25th 04:15 | 60d |

All times UTC. Velero's names are historical and its crons are the truth: see
the terminology warning in [backup-strategy.md](backup-strategy.md).

### Layer 2: ScheduledBackup → VolumeSnapshot

Each cluster has a `backup.volumeSnapshot` stanza (class `longhorn-velero`,
`snapshotOwnerReference: backup`) and a `ScheduledBackup` with
`method: volumeSnapshot` and `backupOwnerReference: self`. A run:

1. CNPG starts an online backup (`pg_backup_start`). It does not request an
   immediate checkpoint by default.
2. It snapshots the instance's PVC through the CSI API. `longhorn-velero` is
   `type: snap`, which is a Longhorn snapshot **inside the source volume** and
   not a copy to a backup store.
3. It stops the backup and stores the `backup_label` and start/end WAL on the
   `Backup` CR and as annotations on the `VolumeSnapshot`.

The ownership chain is
`ScheduledBackup → Backup → VolumeSnapshot → Longhorn snapshot`. The
VolumeSnapshot has the same name as the Backup. Delete the Backup and the rest
cascades away, which is what the retention job relies on.

Three things about where these live, all of which matter for restores:

- **Two-instance clusters snapshot the standby, not the primary.** The cluster
  backup target defaults to `prefer-standby`. On 2026-11-08 the affine
  snapshots came from `affine-cnpg-2`, netbox from `netbox-cnpg-1`, grafana
  from `grafana-cnpg-2` and authentik from `authentik-cnpg-3`.
- **The snapshots die with that volume.** A Longhorn `snap` snapshot is part of
  the volume. The instance PVCs are owned by the `Cluster`, and the `longhorn`
  StorageClass reclaim policy is `Delete`. **Deleting a Cluster deletes its
  PVCs, then its Longhorn volumes, and with them every Layer-2 restore point it
  has.** The `Backup` and `VolumeSnapshot` objects survive (they hang off the
  ScheduledBackup), but they point at nothing. CNPG rebuilding a failed standby
  on a fresh PVC loses that standby's snapshots the same way.
- **None of it leaves the cluster.** Whole-Longhorn loss takes Layer 2 with it.

### Retention: `cnpg-backup-retention`

`ScheduledBackup` has no retention of its own. The `cnpg-backup-retention`
CronJob (`infrastructure/safeqbit-local-hq/configs/cnpg-backup-retention.yaml`,
namespace `cnpg-system`, 05:00 UTC daily) keeps the newest N Backup CRs per
namespace and deletes the rest:

```sh
prune authentik    6    # weekly × ~6 weeks (~900M/snap)
prune affine       7    # daily × 1 week
prune netbox       7    # daily × 1 week
prune monitoring   7    # daily × 1 week (grafana-cnpg)
prune pangolin     4    # weekly × ~4 weeks
```

- pangolin was missing from this list from its creation in August until
  2026-11-08, and had built up 14 Backups by then.
- **The script prunes every Backup in the namespace, hand-made ones
  included.** It sorts by `creationTimestamp` and doesn't look at owners. A
  manual restore point (such as `pangolin-pre-1253-20261029`) is deleted once N
  newer Backups exist: a week for the daily clusters, four weeks for pangolin.
  For anything that must last longer, take a Velero backup.
- A retention change takes effect at the next 05:00 run. To run it now:
  `kubectl -n cnpg-system create job --from=cronjob/cnpg-backup-retention manual-prune-$(date +%s)`.
- It runs as `alpine/k8s:1.34.1` because the script needs GNU `head -n -N`.
  The original `bitnami/kubectl` tag was deleted from Docker Hub, and
  while the job sat in ImagePullBackOff for ~30 days authentik built up 176
  Backups.

**ScheduledBackup crons have six fields** (`sec min hour dom mon dow`). A
five-field `"0 2 * * *"` was read as "minute 2 of every hour" on 2026-05-27 and
made 288 backups in 12 days. Sunday as `0` is fine. authentik and pangolin both
use it.

### Layer 3: Velero → B2

**The Velero backup is the only off-site copy of each database.** Each
namespace's Schedule picks up the CNPG PVCs along with everything else. The
`velero-volume-policy` ConfigMap skips any PVC labelled
`cnpg.io/instanceRole: replica`, so only the PVC that was primary at backup
time gets uploaded. That is a crash-consistent CSI snapshot of a running
primary, and Postgres recovers it like a restart after power loss.

Don't exclude these PVCs to save space. The WAL sizing above already did that.
Until CNPG has a real object-store target (improvement-plan P3.1), there is
nothing else off-site.

### Taking a restore point before risky work

Before an app upgrade that migrates its schema, take a manual Backup (fast,
in-cluster) and, for anything you can't afford to lose, a Velero backup
(off-site):

```bash
kubectl apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pangolin-pre-<version>-<yyyymmdd>
  namespace: pangolin
spec:
  cluster:
    name: pangolin-cnpg
  method: volumeSnapshot
EOF
kubectl -n pangolin get backups.postgresql.cnpg.io pangolin-pre-<version>-<yyyymmdd>   # PHASE completed

velero backup create pangolin-pre-<version>-<yyyymmdd> --from-schedule pangolin-monthly
```

Use the full resource name `backups.postgresql.cnpg.io`. A bare `kubectl get
backup` can resolve to `backups.longhorn.io`.

---

## Restore procedure

> ⚠️ **Not yet drilled.** Neither path below has been run end to end in this
> cluster ([restore-drills.md](restore-drills.md) Drill C). The fields and
> behaviours were checked against the CNPG 1.30.1 API and source and against the
> live objects on 2026-11-08. Do Drill C1 before relying on this in anger.

### Which path

| What happened | Path |
|---|---|
| One instance of a 2-instance cluster died or lost its PVC | **Not a restore.** CNPG fails over and rebuilds the standby from the primary by itself |
| Bad migration, bad data, app bug: the cluster is up but the data is wrong | **A**: Layer-2 snapshot, restored next to the live cluster |
| The cluster is broken but its Longhorn volumes still exist | **A** |
| Longhorn is gone, the volumes are gone, or the cluster was rebuilt | **B**: Velero/B2 |
| The `Cluster` was deleted | **B**. Its Layer-2 snapshots went with it (see above) |

Both paths build a **new cluster with a new name** (convention:
`<cluster>-r1`, then `-r2` and so on) and move the app over to it. That is
deliberate:

- Path A has to leave the old Cluster in place: its volumes hold the snapshot
  being restored. Deleting the old cluster first, which is what the CNPG docs'
  same-name examples imply, deletes the restore point too.
- Path B restores the old PVC under its old name, for example
  `affine-cnpg-1`, which is the PVC name a new `affine-cnpg` would try to create
  for itself.

Restoring under the original name is possible, but it needs a second hop
(restore to `-r1`, then restore `-r1`'s own Backup back to the original name)
and gains nothing but the name. Renaming means changing a few references in Git,
listed under [Point the app at the restored cluster](#point-the-app-at-the-restored-cluster).

### Find the current primary

```bash
kubectl -n <ns> get pod -l cnpg.io/cluster=<cluster>,cnpg.io/instanceRole=primary -o name
kubectl -n <ns> get clusters.postgresql.cnpg.io <cluster> -o jsonpath='{.status.currentPrimary}{"\n"}'
```

Use it for anything that writes, whether that is DDL, an `UPDATE` or a
`CHECKPOINT`. A standby answers reads and fails writes with `cannot execute
UPDATE in a read-only transaction`. That happened on affine in September,
when `affine-cnpg-1` was the replica.

```bash
P=$(kubectl -n affine get pod -l cnpg.io/cluster=affine-cnpg,cnpg.io/instanceRole=primary -o name)
kubectl -n affine exec -it "$P" -c postgres -- psql -d affine
```

### Before either path

1. **Stop Flux from reverting you.** For the app namespaces:
   `flux suspend kustomization apps`. For grafana (Cluster in `configs/`,
   Grafana in the `monitoring` HelmRelease):
   `flux suspend kustomization infrastructure infrastructure-configs infrastructure-controllers`.
   Suspend all three, because the `infrastructure` Kustomization applies the
   same objects as the other two. For apps installed by a HelmRelease, suspend
   that too: `flux suspend helmrelease authentik -n authentik` or
   `flux suspend helmrelease monitoring -n monitoring`.
2. **Decide on the cut-over moment.** Anything the app writes to the old
   database after the snapshot is lost when you switch. If the old database is
   still live and correct up to the incident, take a fresh Backup first (the
   restore-point recipe above), so the bad state can be looked at later.
3. **Mind the 05:00 UTC prune.** If you restore from the oldest Backup in the
   namespace, the retention job can delete it mid-restore. Restore from a newer
   one or finish before 05:00. Once the new cluster is healthy the source
   snapshot is no longer needed.

### A. From a Layer-2 snapshot (Longhorn alive)

1. Pick the Backup and check its snapshot is usable:
   ```bash
   kubectl -n affine get backups.postgresql.cnpg.io --sort-by=.metadata.creationTimestamp
   kubectl -n affine get volumesnapshot affine-cnpg-backup-20261108020000   # READYTOUSE true
   ```
   The VolumeSnapshot has the same name as the Backup. `RESTORESIZE` is the
   minimum `storage.size` for the new cluster.
2. Write the new cluster into Git next to the old one, as a second document in
   the same manifest file. Copy the old spec exactly (same `imageName`, because a
   snapshot is a physical copy and has to come up on the same major version;
   same parameters, storage and `backup` stanza), change the name, and swap
   `bootstrap.initdb` for `bootstrap.recovery`:
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: affine-cnpg-r1
     namespace: affine
   spec:
     instances: 2
     imageName: ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie
     postgresql:
       parameters:
         max_wal_size: "256MB"
         min_wal_size: "80MB"
         wal_keep_size: "128MB"
     bootstrap:
       recovery:
         # Restored from affine-cnpg-backup-20261108020000 on <date>.
         # Create-only: CNPG ignores bootstrap on an existing cluster.
         database: affine
         owner: affine
         volumeSnapshots:
           storage:
             name: affine-cnpg-backup-20261108020000
             kind: VolumeSnapshot
             apiGroup: snapshot.storage.k8s.io
     storage:
       storageClass: longhorn
       size: 5Gi
     backup:
       volumeSnapshot:
         className: longhorn-velero
         snapshotOwnerReference: backup
     affinity:
       topologyKey: kubernetes.io/hostname
     monitoring:
       enablePodMonitor: true
   ```
   - **`database` and `owner` are required here.** Both default to `app`.
     Without them CNPG creates an empty `app` database and `app` user after
     recovery, and the generated secret points at those instead of the
     restored data.
   - Don't add `source` or `externalClusters`. The CNPG docs pair snapshot
     recovery with a WAL archive, but we don't have one. The online backup
     keeps the WAL it needs inside the snapshot, and recovery stops at the end
     of that WAL.
   - `bootstrap.recovery.backup.name: <backup>` also works for a
     volumeSnapshot Backup in the same namespace (CNPG resolves the snapshot
     from it). It cannot be combined with `volumeSnapshots`. Naming the
     snapshot is more explicit, and it is the only option once the Backup CR
     is gone but the VolumeSnapshot is still there.
3. Commit, push, resume only the source of that file
   (`flux resume kustomization apps`), and watch it come up. If it's urgent,
   `kubectl apply -f` the same file and commit it straight after, unchanged.
   ```bash
   kubectl -n affine get pods -l cnpg.io/cluster=affine-cnpg-r1 -w
   # affine-cnpg-r1-1-snapshot-recovery-… (Completed), then affine-cnpg-r1-1, then the standby
   kubectl -n affine get clusters.postgresql.cnpg.io affine-cnpg-r1   # "Cluster in healthy state"
   ```
   The standby may be built by `pg_basebackup` rather than from a snapshot.
   That is slower in general but takes seconds at these sizes.
4. Check the data and the generated credentials before moving anything:
   ```bash
   P=$(kubectl -n affine get pod -l cnpg.io/cluster=affine-cnpg-r1,cnpg.io/instanceRole=primary -o name)
   kubectl -n affine exec "$P" -c postgres -- psql -d affine -Atc 'select count(*) from workspaces'
   # Proves the new affine-cnpg-r1-app password works over the -rw Service:
   URI=$(kubectl -n affine get secret affine-cnpg-r1-app -o jsonpath='{.data.uri}' | base64 -d)
   kubectl -n affine exec "$P" -c postgres -- psql "$URI" -Atc 'select current_user'
   ```
   Spot-check tables that exist in each DB: affine `workspaces`, netbox
   `dcim_device`, authentik `authentik_core_user`, grafana `"user"`, pangolin
   `resources`. Statistics such as `n_live_tup` are reset by recovery, so use
   real `count(*)`.
5. [Point the app at the restored cluster](#point-the-app-at-the-restored-cluster),
   then [clean up](#after-the-cut-over).

### B. From Velero/B2 (Longhorn data gone)

Velero holds one PVC per cluster: whichever was primary at backup time. The plan
is to restore that PVC on its own, then bootstrap a new cluster from it with
`bootstrap.recovery.volumeSnapshots.storage.kind: PersistentVolumeClaim`.
CNPG treats a PVC source like a snapshot: it clones the PVC into the new
instance's volume and leaves the source untouched. The source has no
`backup_label`, so Postgres starts with ordinary crash recovery from the WAL on
the volume.

Restoring the whole namespace, `Cluster` CR included, is not this procedure.
Velero strips the Cluster's status and the PVCs' owner references, and whether
CNPG then adopts a PVC that already exists is not documented, so this page
doesn't rely on it.

1. Find the backup and which PVC it carries (the local `velero` CLI works, so
   there's no need to exec into the Velero pod):
   ```bash
   velero backup get | grep authentik
   velero backup describe authentik-bimonthly-20261105030036 --details   # "CSI Snapshots: authentik/authentik-cnpg-2"
   ```
2. **Clear the way.** After a cluster rebuild, Flux has already created an
   empty `<cluster>` from Git (`bootstrap.initdb`), possibly with a PVC of the
   same name as the one you are about to restore. Velero **skips** objects that
   already exist, so it would restore nothing. With Flux suspended (above) and
   the app scaled to 0, delete the empty cluster and wait for its PVCs to go:
   ```bash
   kubectl -n authentik scale deploy authentik-server authentik-worker --replicas=0
   kubectl -n authentik delete clusters.postgresql.cnpg.io authentik-cnpg
   kubectl -n authentik get pvc -l cnpg.io/cluster=authentik-cnpg -w   # until none
   ```
   Do this only when Longhorn really is gone. Deleting a cluster whose volumes
   still exist destroys its Layer-2 snapshots (use path A instead).
3. Restore just that PVC. The selector matches the label it had at backup time:
   ```bash
   velero restore create authentik-cnpg-pvc-$(date +%Y%m%d) \
     --from-backup authentik-bimonthly-20261105030036 \
     --include-namespaces authentik \
     --include-resources persistentvolumeclaims \
     --selector cnpg.io/cluster=authentik-cnpg,cnpg.io/instanceRole=primary
   velero restore describe authentik-cnpg-pvc-<yyyymmdd> --details   # DataDownload Completed
   kubectl -n authentik get pvc authentik-cnpg-2                     # Bound
   ```
   The data mover needs a temporary Longhorn volume. If its DataDownload sits
   Pending, check the Longhorn scheduling ceiling (maintenance.md).
4. Create the new cluster in Git exactly as in path A step 2, with the
   recovery source pointing at the restored PVC:
   ```yaml
     bootstrap:
       recovery:
         database: authentik
         owner: authentik
         volumeSnapshots:
           storage:
             name: authentik-cnpg-2
             kind: PersistentVolumeClaim
   ```
   `storage.size` must be at least the PVC's size (10Gi for authentik).
   Replace the old cluster's document with this one rather than adding it
   alongside. If the old `authentik-cnpg` stays in Git, Flux recreates it empty
   on resume.
5. Commit, resume, and watch and verify as in path A steps 3–4.
6. Point the app at it (below). Then **delete the restored source PVC**
   (`kubectl -n authentik delete pvc authentik-cnpg-2`). Nothing owns it, and
   it still carries `cnpg.io/instanceRole: primary`, so the next Velero run
   would upload it alongside the real one.

NFS-backed app data (netbox media, authentik media/templates) is not in Velero
at all. It needs the copy-back step in
[backup-strategy.md](backup-strategy.md) → "NFS volume data after a Velero
restore".

### Point the app at the restored cluster

The new cluster brings its own `<new>-rw` Service and `<new>-app` Secret with a
**new password**. CNPG applies it to the restored owner role once recovery
finishes (the `current_user` check in path A step 4 proves it). Anything that names the old cluster has to change, in Git:

| Cluster | File(s) | What to change |
|---|---|---|
| `affine-cnpg` | `apps/safeqbit-local-hq/affine/07-affine.yaml` | `wait-for-db` init: `pg_isready -h affine-cnpg-rw` and its `PGPASSWORD` secretKeyRef. `affine-migration` init and the server: `DATABASE_URL` secretKeyRef (`affine-cnpg-app`, key `uri`) |
| `netbox-cnpg` | `apps/safeqbit-local-hq/netbox/06-netbox-web.yaml`, `07-worker.yaml`, `08-manage.yaml` | `DB_HOST` value, `DB_PASSWORD` secretKeyRef, and the `pg_isready -h` / `PGPASSWORD` in the wait-for-db inits |
| `authentik-cnpg` | `apps/safeqbit-local-hq/authentik/07-helmrelease.yaml` | `authentik.postgresql.host` and the `AUTHENTIK_POSTGRESQL__PASSWORD` secretKeyRef |
| `grafana-cnpg` | `infrastructure/safeqbit-local-hq/controllers/monitoring.yaml` | `grafana.ini.database.host` (`grafana-cnpg-rw.monitoring:5432`) and `envValueFrom.GF_DATABASE_PASSWORD` |
| `pangolin-cnpg` | `apps/safeqbit-local-hq/pangolin/08-pangolin-deployment.yaml` | `POSTGRES_CONNECTION_STRING` secretKeyRef (`pangolin-cnpg-app`, key `uri`) |

In the same commit:

- Point the namespace's `ScheduledBackup` `spec.cluster.name` at the new
  cluster, or Layer 2 stops.
- Nothing else is keyed on the cluster name: the retention job prunes by
  namespace, and the Velero schedules select by namespace.

Then resume Flux (every Kustomization and HelmRelease you suspended), let the
app roll, and check it end to end: log in, open something recent.

### After the cut-over

Once the app has run on the new cluster for a day or so:

1. Remove the old `Cluster` document from Git, merge, and confirm Flux has
   applied it. All Kustomizations here are `prune: false`, so removing it from
   Git deletes nothing. That is the safe order: merge the removal first, then
   delete by hand. Deleting first and merging later lets Flux recreate the
   cluster. That is how guacamole came back after its first decommission
   (2026-08-29, deleted again 2026-09-13).
2. `kubectl -n <ns> delete clusters.postgresql.cnpg.io <old-cluster>`. Its
   PVCs, secrets, PDBs and **its Layer-2 snapshots** go with it. Its old
   `Backup`/`VolumeSnapshot` objects stay behind pointing at nothing, and the
   retention job prunes them over the next N days.
3. Take a Velero backup of the namespace (`--from-schedule`), so B2 has a copy
   of the restored state before the next scheduled run.

### Gotchas

- **A cluster with one instance blocks drains.** The `<cluster>-primary` PDB
  allows 0 disruptions. On a 2-instance cluster CNPG switches the primary over
  ahead of a drain, so the drain proceeds. On a 1-instance cluster (pangolin,
  and any `-r1` you created with `instances: 1`) the drain retries forever.
  Delete the pod by hand
  (maintenance.md quick path) and accept about a minute of DB downtime.
- **Writes need the primary.** See [Find the current primary](#find-the-current-primary).
  `<cluster>-1` is the replica on netbox today and doesn't exist at all on
  authentik.
- **The CNPG PVC in Velero is the only off-site DB copy.** Layer 2 never leaves
  Longhorn. Don't exclude the CNPG PVCs from Velero, and don't count Layer-2
  snapshots as a backup if the Cluster object is about to be deleted.
- **Deleting a Cluster deletes its restore points.** PVC (owned by the
  Cluster) → `Delete` reclaim → Longhorn volume → its `snap` snapshots.
- **No point-in-time recovery.** No WAL archive, so `recoveryTarget` has nothing
  to replay. The newest Layer-2 snapshot is at most a day old (a week for
  authentik and pangolin). The newest Velero copy is up to half a month old (a
  month for pangolin).
- **Same major version only.** A snapshot or PVC restore needs the
  `imageName` the data was written with (PG 18 today).
- **Velero-restored SealedSecret-backed Secrets are orphaned.** If you restored
  a whole namespace instead of one PVC, see restore-drills.md Drill B for the
  adoption fix.
- **Velero restores of pre-2026-07-18 backups** can bring back stale
  cert-manager `CertificateRequest`s that silently stall the next renewal.
  Newer backups exclude them. For older ones, add
  `--exclude-resources certificaterequests.cert-manager.io,orders.acme.cert-manager.io,challenges.acme.cert-manager.io`.

---

## Drains and the primary PDB

Every cluster has a `<cluster>-primary` PDB (`minAvailable: 1` on the primary)
that never allows an eviction. CNPG handles this for 2-instance clusters: when
the primary's node is drained, it switches the primary over first, and the
drain then evicts what is now a replica. For `pangolin-cnpg` there is nothing to
switch to. If `pangolin-cnpg-1` is on the node, delete the pod first, wait for
it to come back 1/1 on another node, then drain. That is step 2 of the quick
path in [maintenance.md](maintenance.md).

---

## History

### Grafana: why SQLite on NFS failed (2026-05)

Grafana 13's "unified storage" runs an embedded API server that keeps dashboards
and other resources as Kubernetes-style objects, with background jobs
(dashboard cleanup, secret lease management, resource version sync) hitting the
database every 30 seconds. SQLite locks with `fcntl`, and file locks over NFS
are slow and unreliable. Add dashboard queries and auth-session lookups and the
writers deadlocked, the same way every time:

```
CNPG dashboard open (from=now-7d, refresh=30s)
  → each refresh fires ~8 Prometheus queries, each taking 4+ min
  → the next refresh fires before they finish; ~100 goroutines pile up
  → auth-session lookups queue behind them; SQLite lock contention spikes
  → SQLITE_BUSY → Grafana unresponsive → liveness probe kills the pod
```

The fix was to move to Postgres on CNPG, where concurrent connections just
queue. Dashboards and datasources are provisioned from ConfigMaps, so nothing
needed migrating: Grafana built a fresh schema on first boot. What the DB holds
now (users, sessions, alert state) is why `grafana-cnpg` gets daily snapshots
and 2 instances, even though it once had neither.

### Vaultwarden: SQLite → Postgres, still not done

Vaultwarden supports Postgres through `DATABASE_URL`, and that is the only route
to running more than one replica. It is still on SQLite (see
[vaultwarden.md](vaultwarden.md)). If it is ever done:

1. Create a `vaultwarden-cnpg` Cluster in the `vaultwarden` namespace.
2. Start Vaultwarden with `DATABASE_URL` pointing at it **and** the SQLite PVC
   still mounted.
3. Migrate SQLite → Postgres. The admin panel needs `ADMIN_TOKEN`, which
   is already in `vaultwarden-secrets`. Check the current upstream method:
   the wiki's migration guide uses `pgloader`.
4. Verify, then drop the SQLite volume.

It's a password manager. Do it in a planned window, with a fresh Velero backup
confirmed beforehand and an export of the vault on hand.

### Earlier state

When this page was first written (2026-05-15), authentik lived in a namespace
called `authentik01`, every other cluster had `instances: 1`, and Grafana had no
ScheduledBackup. The move to 2 instances was improvement-plan P1.3
(2026-05-28). Daily retention was cut from 30 to 7 on 2026-07-21. The WAL
sizing landed on 2026-08-17. The `imageName` pin and the move to PG 18.6
came in the October 2026 wave.
