# Velero PartiallyFailed Backups — Triage & Cleanup

**Cluster:** safeqbit-local-hq
**Last reviewed:** 2026-07-06 (written after the 2026-07-06 cleanup session)

Runbook for when `VeleroBackupPartiallyFailed` (and its usual companion
`KubeJobFailed` on kopia maintenance jobs) fires: how to find which backups
failed and why, how to clear the alert, and how to safely prune the bad
backup objects from the cluster **and** B2.

Context: the 2026-07-03 volume-policy rework ([backup-strategy.md](backup-strategy.md))
fixed the systemic cause of a month of partial failures (Velero trying to
CSI-snapshot NFS volumes → `unable to get valid VolumeSnapshotter for
"velero.io/csi"`). Everything after 07-03 runs clean; this doc is the
procedure used to triage and clean up the leftovers, kept generic for next
time.

> **No local `velero` CLI.** The workstation doesn't have it — every `velero`
> command below runs inside the server pod:
> `kubectl exec -n velero deploy/velero -- /velero <args>`

---

## 1. Triage — what exactly is firing?

The alert summary (Slack) only gives counts. Get the label sets:

```bash
kubectl port-forward -n monitoring svc/alertmanager-operated 19093:9093 &
curl -s 'http://localhost:19093/api/v2/alerts?filter=alertname=~%22Velero.*%7CKubeJob.*%22' | jq -r \
  '.[] | .labels.alertname + " | " + (.labels.schedule // .labels.job_name // "")'
kill %1
```

Then list the backup objects and spot the non-green ones:

```bash
kubectl get backups.velero.io -n velero --sort-by=.metadata.creationTimestamp \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,ERRORS:.status.errors,WARNINGS:.status.warnings,START:.status.startTimestamp \
  | grep -v Completed
```

Cross-check against the live schedules — a firing `schedule` label that no
longer exists in `kubectl get schedules.velero.io -n velero` is an **orphan**
(e.g. `sra-dev-demo-bimonthly` after the rename to `guacamole-bimonthly`)
and needs special handling (§4).

## 2. Root cause — read the backup logs

`.status.failureReason` on the Backup CR is usually empty for partial
failures; the real errors are in the log file in B2. Pull them through the
server pod:

```bash
kubectl exec -n velero deploy/velero -- /velero backup logs <backup-name> \
  | grep -i 'level=error' | sed 's/time="[^"]*" //' | sort -u
```

Known signatures:

| Error | Meaning | Fix |
|---|---|---|
| `unable to get valid VolumeSnapshotter for "velero.io/csi"` on an NFS PV/PVC | Backup tried to CSI-snapshot an NFS volume (no snapshotter exists for it) | Already fixed: `velero-volume-policy` ConfigMap skips all NFS volumes since 2026-07-03. Failure predates the fix → just clear (§3) |
| kopia repo errors / failed `*-kopia-maintain-job` pods | Kopia repo for that namespace broken | See maintenance job section (§5); repo re-init was needed once (monitoring, 2026-06-28) |

Also check pod-volume backups if errors mention uploads:

```bash
kubectl get podvolumebackups.velero.io -n velero -l velero.io/backup-name=<backup-name> \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,VOL:.spec.volume | grep -v Completed
```

## 3. Clearing the alert — how the rule actually works

The rule is **not** "last backup status". It is:

```
increase(velero_backup_partial_failure_total{schedule!=""}[16d]) > 0
  unless on (schedule) (time() - velero_backup_last_successful_timestamp) < 16*86400
```

Consequences:

- A partial failure keeps the alert firing for **16 days** after it happened,
  *unless* a newer **success** on the same schedule suppresses it.
- **Restarting Velero does NOT clear it** (tried 2026-07-06 — the counter
  history lives in Prometheus, not the pod).
- Weeklies usually self-heal before anyone notices; **bimonthly schedules can
  fire for 2+ weeks** waiting for their next run.

So to clear a firing schedule early, give it a success — trigger a run from
the schedule (inherits filters, TTL, volume policy):

```bash
kubectl exec -n velero deploy/velero -- /velero backup create \
  --from-schedule=<schedule-name> <schedule-name>-manual-$(date +%Y%m%d)
```

Watch it (phases: `New → InProgress → WaitingForPluginOperations →
Finalizing → Completed`; data-mover uploads happen during
WaitingForPluginOperations):

```bash
kubectl get backup.velero.io -n velero <name> \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,ERRORS:.status.errors,WARNINGS:.status.warnings
kubectl get datauploads.velero.io -n velero -l velero.io/backup-name=<name>
```

Alert clears on the next Prometheus eval (~1 min) after `Completed`.

## 4. Orphaned schedules (renamed/deleted)

If the schedule no longer exists there is nothing to succeed, so the alert
fires until the failure ages out of the 16-day window. Don't fight it —
**silence it** with an expiry set to the age-out date (failure time + 16d):

```bash
kubectl port-forward -n monitoring svc/alertmanager-operated 19093:9093 &
curl -s -XPOST http://localhost:19093/api/v2/silences -H 'Content-Type: application/json' -d '{
  "matchers": [
    {"name":"alertname","value":"VeleroBackupPartiallyFailed","isRegex":false,"isEqual":true},
    {"name":"schedule","value":"<orphaned-schedule>","isRegex":false,"isEqual":true}
  ],
  "startsAt": "<now RFC3339>",
  "endsAt": "<failure-time + 16d>",
  "createdBy": "fadi",
  "comment": "Schedule renamed/removed; failure ages out of the 16d alert window on <date>."
}'
kill %1
```

(2026-07-06 example: `sra-dev-demo-bimonthly` failed 06-23, silenced until
07-09.)

## 5. Companion alert: KubeJobFailed on kopia maintain jobs

Velero runs `<ns>-default-kopia-maintain-job-*` Jobs in the `velero`
namespace. Failed ones **linger forever** and kube-state-metrics keeps
reporting them even after a later run succeeds. Check the newest run for the
same repo is `Complete`, then delete the failed Job objects:

```bash
kubectl get jobs -n velero --sort-by=.metadata.creationTimestamp | grep -E 'kopia-maintain'
kubectl get backuprepositories.velero.io -n velero        # repo should be Ready
kubectl delete job -n velero <failed-job-1> <failed-job-2>
```

Alert clears immediately.

## 6. Pruning PartiallyFailed backup objects

Why bother: they clutter the `/cluster backups` report for their whole TTL
(weeklies = 180 days), they are **imperfect restore points** (missing the
volumes that errored), and they hold B2 space.

**Safety gate — check before every delete:** the workload must have a newer
`Completed` backup. List per app and eyeball:

```bash
kubectl get backups.velero.io -n velero --sort-by=.metadata.creationTimestamp \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,START:.status.startTimestamp
```

**Delete through Velero, never `kubectl delete backup`.** A plain CR delete
leaves the data in B2 and the backup-sync controller resurrects the CR from
the bucket on its next sync. The Velero delete removes bucket data + CR +
associated snapshots:

```bash
kubectl exec -n velero deploy/velero -- /velero backup delete <backup-name> --confirm
```

Batch form (as run on 2026-07-06 for all 17 leftovers):

```bash
for b in <name1> <name2> ...; do
  kubectl exec -n velero deploy/velero -- /velero backup delete $b --confirm
done
```

Deletion is async (a DeleteBackupRequest is processed by the server). Verify:

```bash
# should return nothing once processed
kubectl get backups.velero.io -n velero -o custom-columns=NAME:.metadata.name,PHASE:.status.phase | grep -v Completed
```

## 7. Post-cleanup checklist

- [ ] `kubectl get backups.velero.io -n velero | grep -v Completed` → empty
- [ ] Prometheus: `ALERTS{alertname=~"VeleroBackupPartiallyFailed|KubeJobFailed",alertstate="firing"}` → only silenced orphans, if any
- [ ] Every workload still has ≥1 `Completed` backup (compare against the schedule table in [backup-strategy.md](backup-strategy.md))
- [ ] Next scheduled runs come back green (check the morning after the next 03:00 window)

## Session log

- **2026-07-06:** 4× `VeleroBackupPartiallyFailed` (photoprism/monitoring/
  guacamole/sra-dev-demo bimonthlies, all pre-07-03 NFS snapshot errors) +
  2× `KubeJobFailed` (stale monitoring kopia-maintain jobs from the 06-28
  repo re-init). Cleared guacamole/monitoring/photoprism with manual
  from-schedule runs (all `Completed`, 0 errors — confirmed the NFS-skip
  policy on bimonthlies), silenced orphaned sra-dev-demo until 07-09,
  deleted the 2 stale jobs, then pruned all 17 PartiallyFailed backups
  (2026-05-14 → 2026-06-28) from cluster + B2 after verifying newer clean
  backups per workload. Inventory is now Completed-only.
