# Backup Strategy

**Cluster:** safeqbit-local-hq
**Last reviewed:** 2026-05-28

Single source of truth for what's backed up, where it goes, how long it's kept, and how to verify or restore. Reflects the live state after the 2026-05-28 rewrite (P2.1 + P2.2 from [improvement-plan.md](improvement-plan.md)).

---

## Three-layer architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 3 — Velero schedules → Backblaze B2  (off-cluster, off-site)  │
│   Coverage: full namespace (manifests, secrets, PVCs via CSI/kopia) │
│   Cadence: weekly for high-churn apps, bi-monthly for others        │
│   Retention: 180 days (TTL on backups), 7d (kopia repo maintenance) │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ data mover offloads snapshots
                                  │
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 2 — CNPG ScheduledBackup → Longhorn VolumeSnapshot            │
│   Coverage: PostgreSQL clusters only (affine, netbox, grafana,      │
│             authentik). Triggered with pg CHECKPOINT for consistency │
│   Cadence: daily (high-value) or weekly (low-churn)                 │
│   Retention: enforced by daily CronJob in cnpg-system               │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ K8s VolumeSnapshot → CSI sidecar
                                  │
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 1 — Longhorn snapshots (in-cluster, on the volume itself)     │
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
| **05:00** | `cnpg-backup-retention` | CronJob (kubectl) | daily | all 4 CNPG clusters | (prune) |

**Day-of-month layout** for bi-monthly Velero jobs (only one workload to B2 per day, avoids B2 transaction-cap spikes):

```
Day:   1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28
              A     V     M        P        Ph              A     V        M     P     Ph
                    |          (Sundays = affine-weekly,  Wednesdays = netbox-weekly,
                    |                                     interleaved with these days)
```

A=authentik, V=vaultwarden, M=monitoring, P=passzilla, Ph=photoprism

---

## Layer 3 — Velero schedules to B2

All Velero schedules use `snapshotMoveData: true`, which:
1. Takes a CSI snapshot via Longhorn
2. Kopia uploads the snapshot data to B2 (`s3.us-east-005.backblazeb2.com`)
3. Once upload succeeds, the local Longhorn snapshot is removed

| Schedule | Cron | Days | TTL | Namespace |
|---|---|---|---|---|
| `affine-weekly` | `0 3 * * 0` | Sundays 03:00 | 180d | affine |
| `netbox-weekly` | `0 3 * * 3` | Wednesdays 03:00 | 180d | netbox |
| `authentik-bimonthly` | `0 3 5,20 * *` | 5th + 20th 03:00 | 180d | authentik |
| `vaultwarden-bimonthly` | `0 3 7,22 * *` | 7th + 22nd 03:00 | 180d | vaultwarden |
| `monitoring-bimonthly` | `0 3 9,24 * *` | 9th + 24th 03:00 | 180d | monitoring |
| `passzilla-bimonthly` | `0 3 11,26 * *` | 11th + 26th 03:00 | 180d | passzilla |
| `photoprism-bimonthly` | `0 3 13,28 * *` | 13th + 28th 03:00 | 180d | photoprism |

**Source of truth:** `infrastructure/safeqbit-local-hq/configs/velero-schedule-*.yaml`

**Velero BackupStorageLocation** `default`:
- Bucket: `k3s-safeqbit-local-hq-velero`
- Endpoint: `https://s3.us-east-005.backblazeb2.com`
- Credentials: SealedSecret `velero-cloud-credentials`

**Kopia maintenance frequency:** `168h0m0s` (weekly per repo, set in `velero.yaml` HelmRelease values as `defaultRepoMaintainFrequency`). Was hourly until 2026-05-27 when it blew through the B2 free-tier transaction cap.

**Namespaces NOT backed up by Velero:**

| Namespace | Why skipped |
|---|---|
| `cloudflared` | Config only — no PVCs. Tunnels reconstruct from manifests + sealed-secrets via Flux. |
| `longhorn-system`, `cnpg-system`, `cert-manager`, `ingress-nginx`, `sealed-secrets`, `kube-system`, `flux-system`, `nfs-provisioner`, `metallb-system`, `velero` | Platform components — reinstall from Flux + manifests in repo. State (CRs, secrets) lives in apps' namespaces. |

---

## Layer 2 — CNPG ScheduledBackups

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

## Retention enforcement — `cnpg-backup-retention` CronJob

**File:** `infrastructure/safeqbit-local-hq/configs/cnpg-backup-retention.yaml`

CNPG's `ScheduledBackup` CRD has no built-in retention. This CronJob (in the `cnpg-system` namespace) runs daily at **05:00 UTC**, lists Backup CRs per cluster, sorts by creationTimestamp, and `kubectl delete`s everything beyond the newest N. Owner-ref handles VolumeSnapshot + Longhorn cleanup.

```yaml
prune authentik   10    # weekly schedule × ~10 weeks
prune affine      30    # daily × ~1 month
prune netbox      30    # daily × ~1 month
prune monitoring  30    # daily × ~1 month (grafana-cnpg)
```

**To tune retention:** edit the `prune` lines in the script section of `cnpg-backup-retention.yaml` and commit. CronJob picks it up on its next run.

**To run on demand** (after editing retention or to clean up immediately):
```bash
kubectl -n cnpg-system create job --from=cronjob/cnpg-backup-retention manual-prune-$(date +%s)
kubectl -n cnpg-system logs -l job-name=manual-prune-... -f
```

---

## Layer 1 — Longhorn snapshots (manual / ad-hoc)

Not on any schedule directly; populated by:
- Layer 2 (CNPG ScheduledBackups via VolumeSnapshot)
- Layer 3 (Velero CSI snapshots, but those are typically removed immediately after data move to B2)
- Manual snapshots via the Longhorn UI (`https://longhorn.local.safeqbit.com`)

**Hard cap:** 250 snapshots per volume. Auto-cleanup of orphan replicas is enabled (`orphan-resource-auto-deletion: replica-data` in `controllers/longhorn.yaml`). See [maintenance.md](maintenance.md) "Longhorn Orphan Replica Cleanup" for the recovery procedure if you ever hit the cap.

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

> ⚠️ **No restore drill has been run yet.** See P2.3 in [improvement-plan.md](improvement-plan.md). Test before you need it.

### Single-namespace restore from Velero
```bash
# List available backups for a namespace
kubectl -n velero get backups.velero.io | grep <namespace>

# Restore
velero restore create --from-backup <backup-name> \
  --include-namespaces <namespace> \
  --restore-volumes
```

### Single CNPG cluster restore from a Backup CR
The CNPG Backup CR can be referenced in a new Cluster spec via `bootstrap.recovery`. See CNPG docs — destroy the old Cluster first, then create a new one pointing at the desired Backup. For full procedure see `cnpg-strategy.md`.

### Roll back a single PVC to a Longhorn snapshot
Use the Longhorn UI: navigate to Volume → Snapshots → select snapshot → "Revert". Requires the volume to be detached first (scale the owning Deployment to 0). Loses any data written since the snapshot.

---

## Known gaps

- **No off-cluster CNPG backup target.** Layer 2 snapshots live on the same Longhorn volumes as the primary. Whole-Longhorn corruption loses both primary AND backups. Only Layer 3 (Velero → B2) protects against that scenario. See P3.1 in [improvement-plan.md](improvement-plan.md) for the future plan to switch CNPG to plugin-based Barman backups direct to S3.
- **No off-site etcd snapshots.** k3s writes etcd snapshots locally only. See P1.2 in [improvement-plan.md](improvement-plan.md) (currently deferred).
- **Single B2 backend.** B2 outage or transaction-cap exhaustion takes Layer 3 down. Cloudflare R2 was considered as an alternative (10M Class A ops free/month vs B2's 2,500/day) — see P2.2.
- **Restore drills not run.** See P2.3.

> Closed gaps (kept for context):
> - ~~No alerts on backup failures.~~ Closed 2026-05-29: P2.5 added `VeleroBackupFailed`, `VeleroBackupFailureRateHigh`, `LonghornVolumeSnapshotCountHigh`, plus CNPG replication/exporter alerts. See [maintenance.md](maintenance.md#monitoring--alerting).

---

## Change log

| Date | Change |
|---|---|
| 2026-05-27 | CNPG ScheduledBackup cron bug fixed (5-field → 6-field). Velero kopia maintenance bumped from 1h → 168h. Manually pruned 119 stale authentik Backup CRs to dodge 250-snapshot cap. |
| 2026-05-28 | P2.1 + P2.2 — full schedule rewrite. Killed `daily-everything` + `weekly-everything`. Per-workload bi-monthly stagger introduced. TTL uniform 180d. CNPG ScheduledBackups added for netbox-cnpg and grafana-cnpg. `cnpg-backup-retention` CronJob deployed. |
| 2026-05-29 | P2.5 — alerts on Velero backup failures, Longhorn snapshot-count approach to 250 cap, CNPG replication lag/exporter health. Added ServiceMonitors for Longhorn (was unscraped) and Velero (was unscraped). |
