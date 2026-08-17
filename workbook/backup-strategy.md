# Backup Strategy

**Cluster:** safeqbit-local-hq
**Last reviewed:** 2026-05-28

Single source of truth for what's backed up, where it goes, how long it's kept, and how to verify or restore. Reflects the live state after the 2026-05-28 rewrite (P2.1 + P2.2 from [improvement-plan.md](improvement-plan.md)).

---

## Three-layer architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 3 - Velero schedules → Backblaze B2  (off-cluster, off-site)  │
│   Coverage: full namespace (manifests, secrets, PVCs via CSI/kopia) │
│   Cadence: bi-weekly / bi-monthly / monthly per workload            │
│   Retention: per-workload TTL, mostly "keep last 2" (see table)     │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ data mover offloads snapshots
                                  │
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 2 - CNPG ScheduledBackup → Longhorn VolumeSnapshot            │
│   Coverage: PostgreSQL clusters only (affine, netbox, grafana,      │
│             authentik). Triggered with pg CHECKPOINT for consistency │
│   Cadence: daily (high-value) or weekly (low-churn)                 │
│   Retention: enforced by daily CronJob in cnpg-system               │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ K8s VolumeSnapshot → CSI sidecar
                                  │
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 1 - Longhorn snapshots (in-cluster, on the volume itself)     │
│   Coverage: any volume that has a snapshot taken                    │
│   Cadence: only when triggered (CNPG, Velero, or manual)            │
│   Retention: limited to 250 per volume (Longhorn hard cap)          │
└─────────────────────────────────────────────────────────────────────┘
```

Each layer protects against a different failure mode:

| Failure | Recovery layer |
|---|---|
| Accidental row delete / app bug 1h ago | Layer 1 (revert volume to snapshot) |
| Database corruption / PVC lost on a node | Layer 2 (CNPG re-bootstraps from snapshot) |
| Whole cluster lost / site disaster | Layer 3 (Velero restore from B2) |

---

## Schedule timeline (full daily picture)

| Time (UTC) | Trigger | Type | Cadence | Workload(s) | Destination |
|---|---|---|---|---|---|
| **02:00** | `affine-cnpg-backup` | CNPG snapshot | daily | affine-cnpg | Longhorn |
| **02:15** | `netbox-cnpg-backup` | CNPG snapshot | daily | netbox-cnpg | Longhorn |
| **02:30** | `grafana-cnpg-backup` | CNPG snapshot | daily | grafana-cnpg | Longhorn |
| **02:45** | `authentik-cnpg-backup` | CNPG snapshot | weekly Sun | authentik-cnpg | Longhorn |
| **03:00** | `affine-weekly` | Velero | weekly Sun | namespace `affine` | B2 |
| **03:00** | `netbox-weekly` | Velero | weekly Wed | namespace `netbox` | B2 |
| **03:00** | `authentik-bimonthly` | Velero | 5th + 20th | namespace `authentik` | B2 |
| **03:00** | `vaultwarden-bimonthly` | Velero | 7th + 22nd | namespace `vaultwarden` | B2 |
| **03:00** | `monitoring-bimonthly` | Velero | 9th + 24th | namespace `monitoring` | B2 |
| **03:00** | `passzilla-bimonthly` | Velero | 11th + 26th | namespace `passzilla` | B2 |
| **03:00** | `photoprism-bimonthly` | Velero | 13th + 28th | namespace `photoprism` | B2 |
| **03:30** | `weekly-trim` | Longhorn fstrim | weekly Sun | every Longhorn volume | (reclaim) |
| **04:15** | `weekly-snapshot` | Longhorn snapshot, retain 1 | weekly Sun | every Longhorn volume | Longhorn |
| **05:00** | `cnpg-backup-retention` | CronJob (kubectl) | daily | all 5 CNPG clusters | (prune) |

**Day-of-month layout** for bi-monthly Velero jobs (only one workload to B2 per day, avoids B2 transaction-cap spikes):

```
Day:   1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28
              A     V     M        P        Ph              A     V        M     P     Ph
                    |          (Sundays = affine-weekly,  Wednesdays = netbox-weekly,
                    |                                     interleaved with these days)
```

A=authentik, V=vaultwarden, M=monitoring, P=passzilla, Ph=photoprism

---

## Layer 3 - Velero schedules to B2

All Velero schedules use `snapshotMoveData: true`, which:
1. Takes a CSI snapshot via Longhorn
2. Kopia uploads the snapshot data to B2 (`s3.us-east-005.backblazeb2.com`)
3. Once upload succeeds, the local Longhorn snapshot is removed

| Schedule | Cron | Days | TTL | Namespace |
|---|---|---|---|---|
| `affine-biweekly` | `0 3 2,17 * *` | 2nd + 17th 03:00 | 21d (keep 2) | affine |
| `netbox-biweekly` | `0 3 4,19 * *` | 4th + 19th 03:00 | 21d (keep 2) | netbox |
| `authentik-bimonthly` | `0 3 5,20 * *` | 5th + 20th 03:00 | 21d (keep 2) | authentik |
| `pulse-bimonthly` | `0 5 6,21 * *` | 6th + 21st 05:00 | 21d (keep 2) | pulse |
| `vaultwarden-bimonthly` | `0 3 7,22 * *` | 7th + 22nd 03:00 | 180d | vaultwarden |
| `monitoring-bimonthly` | `0 3 9,24 * *` | 9th + 24th 03:00 | 21d (keep 2) | monitoring |
| `guacamole-monthly` | `30 4 10 * *` | 10th 04:30 | 45d (keep 2) | guacamole |
| `passzilla-bimonthly` | `0 3 11,26 * *` | 11th + 26th 03:00 | 28d | passzilla |
| `photoprism-bimonthly` | `0 3 13,28 * *` | 13th + 28th 03:00 | 60d | photoprism |
| `immich-bimonthly` | `30 3 1,16 * *` | 1st + 16th 03:30 | 60d | immich |
| `pangolin-monthly` | `15 4 25 * *` | 25th 04:15 | 60d (keep 2) | pangolin |
| `uptime-kuma-weekly` | `0 4 * * 0` | Sundays 04:00 | 60d | uptime-kuma |
| `home-assistant-biweekly` | `0 6 */14 * *` | 1st + 15th + 29th 06:00 | 28d (~keep last 2) | home-assistant |

**Terminology warning.** This repo uses "bi-monthly" to mean *twice a month*
(~15-day interval), not once every two months. "Bi-weekly" means *every 14
days*. The two are therefore almost the same cadence — 2.2 vs 2.0 runs per
month — despite sounding an order of magnitude apart. Read the cron, not the
name.

**Velero has no "keep N".** Retention is TTL-only. To hold N copies the TTL
must sit strictly between N and N+1 intervals: above N so the Nth-oldest
survives, below N+1 so the one before it expires. A TTL *shorter* than one
interval does not give you "keep 1" safely — it gives you a recurring window
with zero off-site copies, which is why nothing in this table is set that way.

**Postgres IS in these backups.** Every namespace with a CNPG cluster ships its
`*-cnpg-1` PVC in the Velero run. The Layer-2 CNPG ScheduledBackups write
Longhorn VolumeSnapshots, which never leave the cluster — so for affine,
netbox, authentik, grafana, guacamole and pangolin, the Velero copy is the
**only** off-site copy of the database. Do not "optimise" it away before
improvement-plan P3.1 gives CNPG a real Barman/S3 target.

**Source of truth:** `infrastructure/safeqbit-local-hq/configs/velero-schedule-*.yaml`

**Velero BackupStorageLocation** `default`:
- Bucket: `k3s-safeqbit-local-hq-velero`
- Endpoint: `https://s3.us-east-005.backblazeb2.com`
- Credentials: SealedSecret `velero-cloud-credentials`

**Kopia maintenance frequency:** `168h0m0s` (weekly per repo, set in `velero.yaml` HelmRelease values as `defaultRepoMaintainFrequency`). Was hourly until 2026-05-27 when it blew through the B2 free-tier transaction cap.

**NFS volumes are deliberately skipped (2026-07-03).** Every Schedule references
the `velero-volume-policy` ConfigMap (`template.resourcePolicy`), whose single
rule skips any NFS-sourced volume. Rationale:

- NFS has no CSI snapshot support. Before the policy, every backup touching an
  NFS PVC errored (`unable to get valid VolumeSnapshotter`), was marked
  **PartiallyFailed**, and silently shipped **no** NFS data to B2 — the
  "coverage" was an illusion.
- TrueNAS owns that data anyway: ZFS periodic snapshots + replication between
  the two NAS boxes + the NAS's own cloud backup. Duplicating bulk files into
  the B2 free tier adds transactions/cost, not durability.
- Affected data: netbox media/reports/scripts, authentik media/templates,
  guacamole recordings, grafana NFS home (plugins — rebuildable), photoprism
  library (static PVs). All under `10.10.10.5:/mnt/nvme2tb/k8s/pvs` (dynamic)
  or `/mnt/mach2/mach2nas/Media` (static). **Both datasets must stay covered
  by the TrueNAS snapshot/replication/cloud tasks — verify when changing NAS
  config.**

**CNPG replica PVCs are skipped (2026-07-03).** The volume policy skips any
PVC labeled `cnpg.io/instanceRole: replica` — primary and replica are
near-identical, so uploading both doubled every DB backup. The operator keeps
the label current across switchovers; restores bring back the primary and CNPG
rebuilds the replica from it.

**Namespaces NOT backed up by Velero:**

| Namespace | Why skipped |
|---|---|
| `cloudflared` | Config only - no PVCs. Tunnels reconstruct from manifests + sealed-secrets via Flux. |
| `longhorn-system`, `cnpg-system`, `cert-manager`, `ingress-nginx`, `sealed-secrets`, `kube-system`, `flux-system`, `nfs-provisioner`, `metallb-system`, `velero` | Platform components - reinstall from Flux + manifests in repo. State (CRs, secrets) lives in apps' namespaces. |

---

## Layer 2 - CNPG ScheduledBackups

CNPG triggers a `pg_backup_start` CHECKPOINT and emits a Backup CR. The operator creates a K8s VolumeSnapshot, which the CSI sidecar passes to Longhorn for the actual point-in-time copy. **Owner-ref chain:**

```
ScheduledBackup
    └─ Backup (one per scheduled run, named cnpg-backup-YYYYMMDDHHmmss)
           └─ VolumeSnapshot (owner-ref: Backup, via snapshotOwnerReference: backup)
                  └─ Longhorn Snapshot (owner-ref: VolumeSnapshot)
```

Delete the **Backup CR** → garbage collection cascades all the way down. This is what the retention CronJob exploits.

| Cluster | Schedule | Cron (6-field) | Cadence | Retention | Notes |
|---|---|---|---|---|---|
| `affine-cnpg` | `affine-cnpg-backup` | `0 0 2 * * *` | daily 02:00 | 30 | Notes app, high churn |
| `netbox-cnpg` | `netbox-cnpg-backup` | `0 15 2 * * *` | daily 02:15 | 30 | Network inventory, critical |
| `grafana-cnpg` | `grafana-cnpg-backup` | `0 30 2 * * *` | daily 02:30 | 30 | ~15MB DB, tiny snapshots |
| `authentik-cnpg` | `authentik-cnpg-backup` | `0 45 2 * * 0` | weekly Sun 02:45 | 10 | SSO, lower change velocity |

**Source of truth:**
- `apps/safeqbit-local-hq/affine/04-cnpg-scheduled-backup.yaml`
- `apps/safeqbit-local-hq/netbox/04-cnpg-scheduled-backup.yaml`
- `apps/safeqbit-local-hq/authentik/04-cnpg-scheduled-backup.yaml`
- `infrastructure/safeqbit-local-hq/configs/grafana-cnpg-scheduled-backup.yaml`

**⚠️ Cron syntax gotcha:** CNPG ScheduledBackup uses a **6-field cron** (`sec min hour dom mon dow`), not the standard 5-field. We were burned by this on 2026-05-27: `"0 2 * * *"` was misparsed as "every hour at minute 2" and produced 288 backups over 12 days. Always write 6 fields.

---

## Retention enforcement - `cnpg-backup-retention` CronJob

**File:** `infrastructure/safeqbit-local-hq/configs/cnpg-backup-retention.yaml`

CNPG's `ScheduledBackup` CRD has no built-in retention. This CronJob (in the `cnpg-system` namespace) runs daily at **05:00 UTC**, lists Backup CRs per cluster, sorts by creationTimestamp, and `kubectl delete`s everything beyond the newest N. Owner-ref handles VolumeSnapshot + Longhorn cleanup.

```yaml
prune authentik    6    # weekly × ~6 weeks (~900M/snap)
prune affine       7    # daily × 1 week
prune netbox       7    # daily × 1 week
prune monitoring   7    # daily × 1 week (grafana-cnpg)
prune guacamole   10    # weekly × ~10 weeks (tiny ~34M snaps)
```

**Why only 7 dailies (trimmed from 30 on 2026-07-21):** the CNPG databases are
tiny (38–53MB) but Postgres keeps a 35-segment × 16MB = **560MB WAL ring** on
the same data volume. Busy writers (Grafana alert state, AFFiNE sync) rewrite
most of that ring every day, so **each daily snapshot costs ~625MB** of
rewritten WAL blocks regardless of how little real data changed — 30 dailies
held ~18.5G (×3 replicas) per cluster for ~40MB of data. Netbox writes little,
so its dailies are only ~65MB. 7 dailies keep RPO at 24h; anything older is
covered by Velero→B2 (Layer 3). If snapshot cost ever needs to shrink further,
the next lever is lowering `max_wal_size` on these clusters to shrink the WAL
ring itself (reloadable, no restart) — not lowering the retain count more.

**To tune retention:** edit the `prune` lines in the script section of `cnpg-backup-retention.yaml` and commit. CronJob picks it up on its next run.

**To run on demand** (after editing retention or to clean up immediately):
```bash
kubectl -n cnpg-system create job --from=cronjob/cnpg-backup-retention manual-prune-$(date +%s)
kubectl -n cnpg-system logs -l job-name=manual-prune-... -f
```

---

## Layer 1 - Longhorn snapshots (weekly recurring + ad-hoc)

**Since 2026-07-21** (`configs/longhorn-recurring-jobs.yaml`, PR #72) every Longhorn
volume gets two weekly RecurringJobs via the built-in `default` group:

- **`weekly-trim`** (Sun 03:30) — fstrim, so filesystem-freed blocks become holes
  instead of being captured into the next snapshot.
- **`weekly-snapshot`** (Sun 04:15, `retain: 1`) — one fresh snapshot per volume per
  week; doubles as a ≤1-week-old local restore point.

This is space hygiene as much as recovery: Longhorn can only reclaim a removed
snapshot by **folding it into a newer snapshot — never into the live volume head**.
Before these jobs existed, system snapshots left behind by reattaches/rebuilds sat
trapped under the head forever (2026-07-21: a 42.5 GB dead snapshot from the 07-09
HA-cutover reattach on the Prometheus TSDB volume — ~127 GB across 3 replicas).
The weekly snapshot provides the fold target; the job's prune triggers the purge.
Snapshot debt per volume is now bounded at ~1 week of block churn. Manual fold
procedure and mechanics: [maintenance.md](maintenance.md) "Longhorn Snapshot Space
Reclaim".

**Exception — Prometheus TSDB volume (2026-07-21, live-only state):** the TSDB
volume (`pvc-068447de-6a38-47df-ae98-e47f1bf4c173`) is opted **out** of
`weekly-snapshot`. Its metrics are deliberately not backed up anywhere (see the
TSDB note under Layer 3), so a standing retain-1 snapshot would permanently pin
~40 GB ×3 replicas of zero-restore-value data. Instead it relies on trim alone:
`weekly-trim` still runs, and `spec.unmapMarkSnapChainRemoved: enabled` lets the
trim reclaim extents held under removed snapshot chains — no fold target needed.
**This lives only in the Volume CR, not in Git.** If the PVC is ever recreated,
re-apply both by hand:

```bash
kubectl -n longhorn-system label volumes.longhorn.io <pvc-...> \
  recurring-job-group.longhorn.io/default- recurring-job.longhorn.io/weekly-trim=enabled
kubectl -n longhorn-system patch volumes.longhorn.io <pvc-...> \
  --type=merge -p '{"spec":{"unmapMarkSnapChainRemoved":"enabled"}}'
```

> **Never add a `snapshot-delete` recurring task cluster-wide.** It deletes ALL
> snapshot kinds beyond its retain count — including the CSI snapshots backing the
> CNPG scheduled-backup VolumeSnapshots (Layer 2) and manual pre-upgrade snapshots.
> The `snapshot` task's retention is label-scoped to its own snapshots and is safe.

Also populated by:
- Layer 2 (CNPG ScheduledBackups via VolumeSnapshot)
- Layer 3 (Velero CSI snapshots, but those are typically removed immediately after data move to B2)
- Manual snapshots via the Longhorn UI (`https://longhorn.local.safeqbit.com`)

**Hard cap:** 250 snapshots per volume. Auto-cleanup of orphan replicas is enabled (`orphan-resource-auto-deletion: replica-data` in `controllers/longhorn.yaml`). See [maintenance.md](maintenance.md) "Longhorn Orphan Replica Cleanup" for the recovery procedure if you ever hit the cap.

---

## Layer 0 - etcd snapshots to B2 (control plane)

**Added 2026-07-03 (P1.2).** k3s takes an etcd snapshot **weekly (Sunday
00:00 UTC)** on each of the 3 server nodes and — via `etcd-s3` config —
uploads each to B2 as well. Weekly is deliberate: this is the THIRD recovery
layer (git/Flux → Proxmox VM backups of the server VMs → etcd-s3), and the
one irreplaceable item (the sealed-secrets private key) never changes. Note
the VM-backup layer restores etcd awkwardly (single restored VM = stale etcd
member needing manual rejoin; all three from different moments = quorum
mess) — the etcd snapshot stays the clean cluster-state restore path. etcd is the cluster's full state: every object, PVC binding,
and critically the **sealed-secrets private key** (exists only in etcd; without
it every SealedSecret in this repo is undecryptable).

- **Bucket:** `k3s-safeqbit-etcd` (separate from the Velero bucket, with an
  application key scoped to it alone — the Velero credentials cannot touch
  etcd snapshots and vice versa).
- **Config:** `/etc/rancher/k3s/config.yaml` on each server node (root-only,
  0600 — contains the B2 key, deliberately NOT in git). Fields: `etcd-s3: true`,
  endpoint/region `s3.us-east-005.backblazeb2.com`/`us-east-005`, bucket,
  `etcd-s3-folder: snapshots`, `etcd-snapshot-retention: 8` (8 weeks of
  weekly snapshots per node), `etcd-snapshot-schedule-cron: "0 0 * * 0"`.
- **Bucket lifecycle:** "keep prior versions for 30 days" — k3s pruning
  deletes ~3 objects/week; the 30-day hidden tail (~1GB) makes any deletion
  done with the node-side key undoable from the B2 console (account creds,
  not the stolen key) for a month. Steady state ≈ 2GB live + 1GB tail.
- **Verify:** `kubectl get etcdsnapshotfiles | grep s3://` — S3-flavored
  records appear next to the `file://` ones after each scheduled snapshot.
  On-demand test: `sudo k3s etcd-snapshot save` on any server node.
  (The `Unknown flag --tls-san` warning from the snapshot subcommand is
  harmless — it parses config.yaml but only knows snapshot flags.)
- **Restore (full control-plane loss):** on a fresh node,
  `k3s server --cluster-reset --cluster-reset-restore-path=<snapshot-name> \
  --etcd-s3 --etcd-s3-endpoint=... --etcd-s3-bucket=k3s-safeqbit-etcd \
  --etcd-s3-folder=snapshots --etcd-s3-access-key=... --etcd-s3-secret-key=...`
  then rejoin the other servers. See k3s docs "Restoring a Snapshot".

---

## Verification commands

```bash
# List all Velero schedules + last-run status
kubectl -n velero get schedules -o custom-columns=NAME:.metadata.name,SCHEDULE:.spec.schedule,LAST:.status.lastBackup

# List recent Velero backups (last 10)
kubectl -n velero get backups.velero.io --sort-by=.metadata.creationTimestamp | tail -10

# CNPG backups per cluster
kubectl get backups.postgresql.cnpg.io -A --sort-by=.metadata.creationTimestamp

# Backup CR counts per namespace (should plateau near retention values)
for ns in affine authentik netbox monitoring; do
  echo "$ns: $(kubectl -n $ns get backups.postgresql.cnpg.io --no-headers 2>/dev/null | wc -l)"
done

# Longhorn snapshot total
kubectl -n longhorn-system get snapshots.longhorn.io --no-headers | wc -l

# Retention CronJob last run + status
kubectl -n cnpg-system get cronjob cnpg-backup-retention
kubectl -n cnpg-system logs -l job-name --tail=50

# Velero BackupStorageLocation health
kubectl -n velero get bsl
# Phase=Available means B2 credentials + connectivity are good
```

---

## Restoring

> ⚠️ **No restore drill has been run yet.** The drill playbook now exists — [restore-drills.md](restore-drills.md) (P2.3) — but the first run is still pending. Test before you need it.

### Single-namespace restore from Velero
```bash
# List available backups for a namespace
kubectl -n velero get backups.velero.io | grep <namespace>

# Restore
velero restore create --from-backup <backup-name> \
  --include-namespaces <namespace> \
  --restore-volumes
```

### NFS volume data after a Velero restore (manual step!)

Velero restores the PVC *object* but NFS **data** is not in the backup (see
the volume policy above). The nfs-subdir provisioner then creates a **new,
empty** directory for the restored PVC (the path embeds the new PV name), so
the app comes up with blank NFS volumes. To reconnect the data:

```bash
# 1. Old data still lives in the previous subdir on TrueNAS (or in a ZFS
#    snapshot of it): /mnt/nvme2tb/k8s/pvs/<ns>-<pvc-name>-pvc-<old-uid>/
# 2. Find the NEW subdir the provisioner created for the restored PVC:
kubectl get pv $(kubectl -n <ns> get pvc <pvc> -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.nfs.path}{"\n"}'
# 3. On the NAS, copy old contents into the new subdir (rsync -a), then
#    restart the pod. Ownership must match the pod's fsGroup/runAsUser.
```

Static NFS PVs (photoprism/immich media) need no copy — they point at fixed
paths; restore the data in-place from a ZFS snapshot if it was damaged.

### Single CNPG cluster restore from a Backup CR
The CNPG Backup CR can be referenced in a new Cluster spec via `bootstrap.recovery`. See CNPG docs - destroy the old Cluster first, then create a new one pointing at the desired Backup. For full procedure see `cnpg-strategy.md`.

### Roll back a single PVC to a Longhorn snapshot
Use the Longhorn UI: navigate to Volume → Snapshots → select snapshot → "Revert". Requires the volume to be detached first (scale the owning Deployment to 0). Loses any data written since the snapshot.

---

## Known gaps

- **No off-cluster CNPG backup target.** Layer 2 snapshots live on the same Longhorn volumes as the primary. Whole-Longhorn corruption loses both primary AND backups. Only Layer 3 (Velero → B2) protects against that scenario. See P3.1 in [improvement-plan.md](improvement-plan.md) for the future plan to switch CNPG to plugin-based Barman backups direct to S3.
- **Single B2 backend.** B2 outage or transaction-cap exhaustion takes Layer 3 down. Cloudflare R2 was considered as an alternative (10M Class A ops free/month vs B2's 2,500/day) - see P2.2.
- **Restore drills not run.** Playbook drafted 2026-07-05 ([restore-drills.md](restore-drills.md)); first drill still pending. See P2.3.
- **Prometheus TSDB (metrics history) has no off-site copy — deliberate.** Decided 2026-07-03: the 40Gi TSDB never fit the B2 10GiB free tier (its DataUploads had never once succeeded). The volume policy skips volumes ≥20Gi. After a full rebuild Grafana starts with empty graphs that refill over the 30d retention window; dashboards (git), alert rules (git), and Grafana's DB (CNPG + Velero) survive regardless. Revisit if the Velero target ever moves to NAS MinIO (no size pressure there).

> Closed gaps (kept for context):
> - ~~No off-site etcd snapshots.~~ Closed 2026-07-03: P1.2 — k3s `etcd-s3` upload to dedicated `k3s-safeqbit-etcd` B2 bucket with a bucket-scoped key. See "Layer 0" above.
> - ~~No alerts on backup failures.~~ Closed 2026-05-29: P2.5 added `VeleroBackupFailed`, `VeleroBackupFailureRateHigh`, `LonghornVolumeSnapshotCountHigh`, plus CNPG replication/exporter alerts. See [maintenance.md](maintenance.md#monitoring--alerting).
> - ~~Partial backup failures were invisible.~~ Closed 2026-06-28: `velero_backup_last_status` reads 1 even for PartiallyFailed, so `VeleroBackupFailed` missed `monitoring-bimonthly` being broken ~6 weeks. Added `VeleroBackupPartiallyFailed` (self-healing). See [maintenance.md](maintenance.md#velero).
> - ~~No alert on the Longhorn over-provisioning ceiling.~~ Closed 2026-06-28: `LonghornNodeStorageLow` only watches physical usage; the *scheduling* ceiling silently broke all data-mover backups. Added `LonghornNodeSchedulingCeiling`. See [maintenance.md](maintenance.md#longhorn-capacity--disk-expansion).

---

## Change log

| Date | Change |
|---|---|
| 2026-05-27 | CNPG ScheduledBackup cron bug fixed (5-field → 6-field). Velero kopia maintenance bumped from 1h → 168h. Manually pruned 119 stale authentik Backup CRs to dodge 250-snapshot cap. |
| 2026-05-28 | P2.1 + P2.2 - full schedule rewrite. Killed `daily-everything` + `weekly-everything`. Per-workload bi-monthly stagger introduced. TTL uniform 180d. CNPG ScheduledBackups added for netbox-cnpg and grafana-cnpg. `cnpg-backup-retention` CronJob deployed. |
| 2026-05-29 | P2.5 - alerts on Velero backup failures, Longhorn snapshot-count approach to 250 cap, CNPG replication lag/exporter health. Added ServiceMonitors for Longhorn (was unscraped) and Velero (was unscraped). |
| 2026-07-03 | P1.2 closed: etcd snapshots to B2. `etcd-s3` config on all 3 server nodes (rolling k3s restarts, no workload impact; also first live validation of the CoreDNS autoscaler surviving a k3s Addon re-apply). Dedicated bucket `k3s-safeqbit-etcd` + bucket-scoped key. Verified: on-demand snapshot uploaded (~29MB). |
| 2026-07-03 | B2 tuning round 2. (1) immich-bimonthly schedule added — immich had NO backup (plain-Deployment postgres, no CNPG layer; ~470MB) — 1st+16th 03:30, 60d TTL, model-cache PVC excluded via label. (2) CNPG replica PVCs skipped via pvcLabels volume policy (halves DB upload churn). (3) TTLs tuned per workload: 180d→60d for netbox/affine/monitoring/photoprism/uptime-kuma/pulse, 180d→90d authentik, 180d→28d passzilla; vaultwarden stays 180d (tiny + critical), guacamole stays 28d/keep-2. Existing Backup CRs keep their original 180d expirations; only new backups get the shorter TTLs. |
| 2026-07-03 | Prometheus TSDB excluded from Velero (volume policy: skip ≥20Gi; only other ≥20Gi volume is guacamole-recordings, already NFS-skipped). Metrics history = accepted DR loss. Unblocks monitoring-bimonthly going fully green on 07-09 without blowing the B2 10GiB free tier (bucket at ~2.8GiB). Same day: BSL polling cut 1m→1h (B2 Class C cap emails), Velero success notifications + weekly Slack backup digest added, netbox-housekeeping CronJob unwedged (image script never exits; now runs manage.py directly + activeDeadlineSeconds). |
| 2026-07-03 | NFS volume policy. All 10 Velero Schedules now reference the `velero-volume-policy` ConfigMap (skip all NFS volumes). Ends the standing PartiallyFailed status on netbox/authentik/guacamole/monitoring/photoprism backups — NFS data was never actually uploaded, only errored. NFS protection = TrueNAS (ZFS snapshots + inter-NAS replication + NAS cloud backup). Documented the copy-back-into-new-subdir step for NFS data after a Velero restore. Known residual: monitoring Longhorn DataUploads (Prometheus TSDB) still failing — scoping decision pending; grafana-cnpg PVCs still redundantly uploaded by Velero alongside CNPG snapshots. |
| 2026-06-28 | Incident response. (1) `cnpg-backup-retention` was `ImagePullBackOff` ~30d (`bitnami/kubectl:1.34` removed from Docker Hub) → repinned to `alpine/k8s:1.34.1`; ran a manual prune (authentik 176→10). (2) `monitoring-default-kopia` was uninitialized → monitoring DataUploads PartiallyFailed ~6 weeks; fixed by deleting the stale `BackupRepository` CR to force re-init. (3) Longhorn scheduling ceiling (over-provisioning 100%) blocked all data-mover temp volumes after a Prometheus PVC expansion; reclaimed orphaned `sra-dev-demo` ns, then grew each node's sdb 150→250 GiB (`xfs_growfs`). (4) Added `LonghornNodeSchedulingCeiling` + self-healing `VeleroBackupPartiallyFailed` alerts; removed orphan `pangolin-bimonthly`. Full runbooks in [maintenance.md](maintenance.md). |
| 2026-08-17 | B2 tuning round 3 (~31GB source / ~10GB stored → ~8GB source / ~2.7GB stored). (1) **CNPG WAL ring shrunk** — the real cause of the footprint: CNPG defaults `wal_keep_size=512MB` / `max_wal_size=1GB` pinned `pg_wal` at ~561MB on all six clusters while `base/` was only 33–245MB, so ~90% of every CNPG backup was empty WAL. Set to 128MB/256MB fleet-wide (sighup params, no restart). Saves ~400MB per CNPG backup — more than every TTL change combined. (2) Cadence/TTL rework: affine weekly→bi-weekly (2nd+17th, 21d), netbox weekly→bi-weekly (4th+19th, 21d), pulse weekly→bi-monthly (6th+21st, 21d), guacamole bi-monthly→monthly (10th, 45d), pangolin bi-monthly→monthly (25th, 60d), authentik 90d→21d, monitoring 60d→21d — all now "keep last 2". Five schedules renamed to match their cadence; old Schedule CRs hand-deleted (infrastructure-configs has `prune: false`). (3) Stale one-off backups purged (~3.8GB source): netbox-weekly-manual-nfspolicy-test, authentik-postupgrade-20260719c, monitoring-bimonthly-manual-20260706, photoprism-bimonthly-manual-20260706, immich-manual-initial, pulse-pre/post-v6-20260725, and 2 orphaned vaultwarden-frequent backups. (4) `VeleroBackupPartiallyFailed` window 16d→32d so a monthly cadence can still self-heal. (5) Corrected the "Postgres data is NOT in this backup" comment, which was wrong in every schedule file. (6) Also deleted the orphaned `sra-dev-demo-default-kopia` BackupRepository, left behind by the guacamole namespace rename and still being maintained weekly (these CRs are runtime-created by Velero, not in Git). |
| 2026-08-17 | **Correction to the row above.** It recorded `pulse-data` as "grew 227MB→967MB in three weeks … needs retention trimming". That was wrong — it read a one-time step change as a growth trend. The v6.1.1 upgrade on 2026-07-25 took pulse-data from 228MB to 1032MB **in a single day** (v6 introduced the tiered metrics store); since then it has oscillated 947–1133MB and decreased twice. It is at **steady state**, ~875MB live on a 2Gi PVC, with retention pruning every tier hourly. No action needed and none taken. Verified separately that the SQLite files are not bloated: `metrics.db` (544MB) and `unified_resources.db` (176MB) both report a **freelist of 0**, so the space is live rows, not reclaimable free pages — VACUUM would gain nothing. Only `notification_queue.db` is bloated (18.5MB of 23.7MB reclaimable), which is immaterial. Tuning levers if it ever is needed, in upstream's recommended order: lower `pvePollingInterval` (currently 10s, inflates the 2h raw tier), then `metricsRetentionMinuteHours` (24h default = ~60% of all rows) in `/data/system.json`; both need a Pulse restart. Defaults are raw 2h / minute 24h / hourly 7d / daily 90d. |
