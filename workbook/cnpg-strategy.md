# CNPG Strategy & Database Inventory

**Cluster:** safeqbit-local-hq  
**Last updated:** 2026-05-15

---

## Overview

CloudNativePG (CNPG) is the cluster-wide Postgres operator, deployed in `cnpg-system`
and watching all namespaces. Any workload that needs Postgres declares its own `Cluster`
CR in its own namespace — the operator reconciles them all.

The operator itself runs 2 replicas (pod anti-affinity across nodes) for HA. It creates
three Services per cluster automatically:

| Service suffix | Routes to | Used for |
|---|---|---|
| `-rw` | Current primary | All writes, normal app connection |
| `-ro` | Hot standbys only | Read scaling (if instances > 1) |
| `-r` | Any instance | Load-balanced reads |

Credentials are auto-generated per cluster into a `<cluster-name>-app` secret with 11
keys (`username`, `password`, `host`, `uri`, `jdbc-uri`, etc.).

---

## Workload Database Inventory

| Workload | Namespace | DB Engine | Storage | CNPG | Notes |
|---|---|---|---|---|---|
| Authentik | `authentik` | PostgreSQL | Longhorn 10Gi | ✅ `instances: 2` | SSO backend — 2 instances for fast failover |
| Authentik01 | `authentik01` | PostgreSQL | Longhorn 10Gi | ✅ `instances: 2` | Second Authentik node migrated from Docker host 10.10.12.29 |
| Grafana | `monitoring` | PostgreSQL | Longhorn 5Gi | ✅ `instances: 1` | Migrated from SQLite/NFS due to deadlock issue |
| Netbox | `netbox` | PostgreSQL | Longhorn 5Gi | ✅ `instances: 1` | Migrated from Docker/Portainer 2026-05-15 |
| Vaultwarden | `vaultwarden` | SQLite | Longhorn PVC | ⏳ future | See notes below |
| Passzilla | `passzilla` | SQLite | Longhorn PVC | ❌ not planned | Ephemeral utility, no durability need |

---

## Instance Count Rationale

All clusters start at `instances: 1` unless there is a specific reason to go higher.
Longhorn already replicates volume data across 2 nodes, so data is safe even with a
single CNPG instance — a node failure just means a ~60s pod reschedule.

### When to use `instances: 2`

Add a hot standby when the workload is a dependency for other services. If its database
goes down, do other things break too? If yes, the faster CNPG-managed promotion (~10–30s)
is worth the extra storage.

**Authentik is `instances: 2`** because it is the SSO backend for the entire cluster.
When Authentik's database was unavailable during the initial CNPG migration, Grafana
cascaded into SQLite deadlock within minutes. A hot standby reduces that blast radius.

### When `instances: 1` is fine

- **Grafana** — UI tool. 60s downtime is acceptable.
- **Vaultwarden** — Single-user personal vault. 60s downtime is acceptable.
- **Passzilla** — Ephemeral utility. Not worth the storage overhead.

### `instances: 3` — not used

Three instances gives you a primary + 2 standbys, which allows Postgres to elect a new
primary by quorum without manual intervention. In a 3-node homelab this adds cost for
little benefit — CNPG handles promotion without quorum anyway.

---

## Backup Strategy

Every CNPG cluster is paired with a `ScheduledBackup` CR using the `volumeSnapshot`
method and the `longhorn-velero` VolumeSnapshotClass.

**How CNPG VolumeSnapshot backups work:**
1. CNPG issues a `CHECKPOINT` to Postgres — all dirty pages flushed to disk
2. Postgres confirms checkpoint complete
3. CNPG triggers a VolumeSnapshot via the CSI API (Longhorn)
4. The snapshot represents a guaranteed-clean, crash-consistent state
5. A `Backup` object is created; it owns the `VolumeSnapshot` via `snapshotOwnerReference: backup`

This is application-aware — unlike Velero snapshotting the raw PVC without Postgres
knowing. Velero continues to back up everything else in each namespace (media PVCs,
secrets, ConfigMaps) on the biweekly schedule.

| Cluster | Schedule | Method |
|---|---|---|
| `authentik-cnpg` | Daily 01:00 UTC | VolumeSnapshot |
| `grafana-cnpg` | None (no ScheduledBackup) | Velero biweekly covers it |

Grafana has no `ScheduledBackup` because all its dashboards and datasources are
provisioned from GitOps ConfigMaps — if the database were lost, Grafana reinitialises
cleanly from scratch. No point snapshotting it.

---

## Grafana: Why SQLite/NFS Failed

Grafana 13 introduced "unified storage" — an embedded Kubernetes-like API server that
manages dashboards, playlists, and other resources as K8s objects. This runs several
background jobs against the database every 30 seconds:

- Dashboard cleanup (`dashboard-service`)
- Secret lease management
- Resource version sync

SQLite uses file-level locking (fcntl). Over NFS, file locks are unreliable and slow.
With 30s background jobs, plus user dashboard queries, plus auth session lookups, the
concurrent SQLite writers deadlock. The sequence every time:

```
CNPG dashboard open (from=now-7d, refresh=30s)
  → each refresh fires ~8 Prometheus queries
  → queries take 4+ minutes (sparse data over a 7-day range)
  → 30s refresh fires again before queries finish
  → goroutine pool fills with ~100 pending requests
  → auth session lookups queue behind query goroutines
  → SQLite lock contention spikes (NFS makes it worse)
  → context canceled cascades
  → SQLITE_BUSY → Grafana fully unresponsive → liveness probe kills pod
```

**Fix:** PostgreSQL via CNPG. Concurrent connections queue properly with no file locking.

**Secondary note:** The CNPG dashboard's `from=now-7d` queries are genuinely slow when
CNPG metrics are new (Prometheus scans mostly-empty time ranges). Use `now-1h` or
`now-3h` on that dashboard until a few days of data have accumulated.

---

## Vaultwarden: Future Migration Notes

Vaultwarden supports PostgreSQL via the `DATABASE_URL` environment variable. The
deployment comment already flags this as the path to HA. Migration steps when ready:

1. Set `ADMIN_TOKEN` in Vaultwarden's sealed secret (needed to access `/admin`)
2. Create a `vaultwarden-cnpg` Cluster in the `vaultwarden` namespace
3. Start Vaultwarden with the new `DATABASE_URL` pointing at CNPG **and** the old
   SQLite PVC still mounted
4. Use the Vaultwarden admin panel → Diagnostics → **Migrate Database** to copy
   SQLite → PostgreSQL in one operation
5. Verify data in PostgreSQL, then remove the `DATABASE_URL` reference to SQLite
   and the old PVC

**Do not do this casually** — this is someone's password manager. Plan a maintenance
window and have a Velero backup confirmed before starting.

---

## Key Files

```
apps/safeqbit-local-hq/authentik/
├── 03-cnpg-cluster.yaml           # Cluster CR, instances: 2, 10Gi
└── 04-cnpg-scheduled-backup.yaml  # Daily 01:00 UTC VolumeSnapshot

apps/safeqbit-local-hq/netbox/
└── 03-cnpg-cluster.yaml           # Cluster CR, instances: 1, 5Gi

infrastructure/safeqbit-local-hq/configs/
└── grafana-cnpg.yaml              # Cluster CR, instances: 1, 5Gi

infrastructure/safeqbit-local-hq/controllers/
├── cnpg.yaml                      # Operator HelmRelease, replicaCount: 2
└── monitoring.yaml                # Grafana wired to grafana-cnpg-rw via GF_DATABASE_*
```
