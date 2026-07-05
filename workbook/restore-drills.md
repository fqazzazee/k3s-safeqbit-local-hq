# Restore Drills

**Cluster:** safeqbit-local-hq
**Created:** 2026-07-05 (P2.3 in [improvement-plan.md](improvement-plan.md))
**Cadence:** one drill per quarter, rotating A → B → C → D. Log every run in the [Results log](#results-log), even (especially) failures.

Backups that haven't been tested aren't backups. The cluster has four layers
(etcd→B2, Longhorn snapshots, CNPG ScheduledBackups, Velero→B2 — see
[backup-strategy.md](backup-strategy.md)) and none has been exercised
end-to-end. Each drill below validates one layer, ordered easiest-first.

**Ground rules**

- Run the **non-destructive variant first**. Only do the destructive variant
  once the non-destructive one has passed at least once.
- Drill on the designated low-stakes app. Don't improvise a drill on
  vaultwarden or authentik.
- Before any destructive step, take an ad-hoc safety net: a manual Longhorn
  snapshot (Drill A/B) or an on-demand Velero backup
  (`velero backup create pre-drill-<app>-$(date +%Y%m%d) --include-namespaces <ns>`).
- Suspend Flux for the app **only** when the drill deletes/recreates objects
  Flux owns, and remember `flux resume` after (the `FluxSuspended` alert will
  nag daily after 4h — that's working as intended).
- Timebox: if a drill exceeds 2× its estimate, stop, roll back via the safety
  net, and log what blocked it. A blocked drill is a successful finding.

---

## Drill A — Single-PVC rollback from a Longhorn snapshot

**Layer tested:** 1 (Longhorn snapshot + revert)
**Guinea pig:** `uptime-kuma` (2Gi Longhorn RWO, SQLite — self-contained, low blast radius)
**Estimate:** ~30 min
**Simulates:** app bug / accidental data change an hour ago

1. Write a marker: in the Uptime Kuma UI, create a throwaway monitor named
   `DRILL-DELETE-ME`.
2. Take a manual snapshot: Longhorn UI (`longhorn.local.safeqbit.com`) →
   Volume → (the uptime-kuma volume) → Take Snapshot, name it `drill-YYYYMMDD`.
3. Write post-snapshot data: rename the marker monitor to
   `DRILL-AFTER-SNAPSHOT`.
4. Detach: `kubectl -n uptime-kuma scale deploy uptime-kuma --replicas=0`
   (wait for the volume to show Detached in Longhorn).
5. Revert: Longhorn UI → Volume → Snapshots → select `drill-YYYYMMDD` →
   Revert.
6. Reattach: scale back to 1, wait Ready.

**Pass criteria:** the monitor is named `DRILL-DELETE-ME` again (post-snapshot
rename rolled back), all other monitors/history intact, no volume degraded.
**Cleanup:** delete the marker monitor and the `drill-YYYYMMDD` snapshot.

> Note Flux will not fight the scale-to-0 (replicas isn't enforced fast), but
> if a reconcile races you, briefly `flux suspend kustomization apps`.

---

## Drill B — Namespace restore from Velero/B2

**Layer tested:** 3 (Velero + kopia data-mover + B2), the site-disaster layer
**Guinea pig:** `passzilla` (single Longhorn PVC, no CNPG, no NFS, 28d TTL — the simplest real workload)
**Estimate:** ~1–2 h

### B1 — non-destructive (restore into a scratch namespace)

1. Write a marker: create a password push in the pwpush UI, note its URL.
   Wait until after the next scheduled backup (11th/26th) **or** trigger one:
   `velero backup create drill-passzilla-$(date +%Y%m%d) --include-namespaces passzilla`
   and wait for `Completed` (not PartiallyFailed) in
   `kubectl -n velero get backup drill-... -o jsonpath='{.status.phase}'`.
2. Restore with a namespace mapping so the live app is untouched:
   ```bash
   velero restore create drill-restore-$(date +%Y%m%d) \
     --from-backup drill-passzilla-YYYYMMDD \
     --namespace-mappings passzilla:passzilla-drill
   ```
3. Watch: `velero restore describe drill-restore-... --details`. The
   data-mover DataDownload needs a temp Longhorn volume — if it sits Pending,
   check the Longhorn scheduling ceiling (see maintenance.md).
4. The restored Deployment will collide on the Ingress host — that's fine;
   verify via port-forward instead:
   `kubectl -n passzilla-drill port-forward deploy/passzilla 8080:5100`.

**Pass criteria:** pod Ready in `passzilla-drill`, the marker push from step 1
exists in the restored copy (proves PVC **data** came back from B2, not just
manifests).
**Cleanup:** `kubectl delete ns passzilla-drill` and delete the drill backup
(`velero backup delete drill-passzilla-...`).

### B2 — destructive (full delete + restore, only after B1 has passed)

1. Confirm a fresh `Completed` backup exists (step B1.1).
2. `flux suspend kustomization apps` (stop Flux recreating the namespace
   mid-restore with empty PVCs).
3. `kubectl delete ns passzilla` — wait for full termination.
4. `velero restore create --from-backup <backup> --include-namespaces passzilla`.
5. Verify the marker push at `passzilla.local.safeqbit.com` (SealedSecret,
   cert, ingress all restored — the TLS secret comes from the backup;
   cert-manager may reissue, both are fine).
6. `flux resume kustomization apps` and confirm the next reconcile is a
   no-op (`flux get kustomization apps`).

**Gotchas already known** (from backup-strategy.md — check them off during the drill):
- NFS PVC data is **not** in Velero backups (volume policy). Passzilla has
  none, but any drill on netbox/authentik/guacamole needs the manual NFS
  copy-back step documented in backup-strategy.md → "Restoring".
- Restored PVCs get new PV names; Longhorn shows them as new volumes (old
  replicas become orphans — auto-cleanup handles them).

---

## Drill C — CNPG cluster restore from a Backup CR

**Layer tested:** 2 (CNPG ScheduledBackup → VolumeSnapshot → recovery bootstrap)
**Guinea pig:** `affine-cnpg` (daily backups, 30 retained; affine tolerates brief DB pauses)
**Estimate:** ~1 h

### C1 — non-destructive (recovery clone alongside the live cluster)

1. Pick the newest completed Backup CR:
   `kubectl -n affine get backups.postgresql.cnpg.io --sort-by=.metadata.creationTimestamp | tail -3`
2. Create a **new, differently-named** cluster bootstrapped from it (do NOT
   commit this to git — it's throwaway; apply directly):
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: affine-cnpg-drill
     namespace: affine
   spec:
     instances: 1
     storage:
       size: 5Gi            # match the live cluster's storage size
       storageClass: longhorn
     bootstrap:
       recovery:
         backup:
           name: <backup-cr-name>
   ```
3. Wait for `affine-cnpg-drill-1` Ready
   (`kubectl -n affine get cluster affine-cnpg-drill`).
4. Compare row counts against the live primary:
   ```bash
   for c in affine-cnpg affine-cnpg-drill; do
     kubectl -n affine exec ${c}-1 -c postgres -- \
       psql -U postgres -d app -tAc \
       "select count(*) from pg_stat_user_tables;"
   done
   ```
   Then spot-check a busy table's count in both. Counts should match as of
   the backup time (drill clone may be slightly behind live — expected).

**Pass criteria:** drill cluster reaches Ready from the snapshot alone and
data is present and plausible.
**Cleanup:** `kubectl -n affine delete cluster affine-cnpg-drill` (its PVCs
cascade).

### C2 — destructive (replace the live cluster; only after C1 passes, and take a fresh Backup CR first)

Full procedure per [cnpg-strategy.md](cnpg-strategy.md): suspend Flux for
apps, scale the affine app to 0, delete the live `affine-cnpg` Cluster,
recreate it with the same name + `bootstrap.recovery` from the chosen Backup,
wait Ready (the app secret `affine-cnpg-app` is re-minted — password changes;
the app consumes it via secretKeyRef so a pod restart picks it up), scale the
app up, verify login + recent docs, resume Flux.
**Watch for:** the git manifest for `affine-cnpg` has no `bootstrap.recovery`
— Flux resume must NOT revert the Cluster spec into a re-init. CNPG ignores
bootstrap changes after creation (bootstrap is create-only), so resume is
safe — but verify `kubectl -n affine get cluster affine-cnpg` stays healthy
for 10 min after resume, and record the observation.

---

## Drill D — Full DR rebuild (tabletop first, live later)

**Layer tested:** 0 + everything (etcd→B2, git+Flux, Velero) — the "all three VMs are gone" scenario
**Estimate:** tabletop ~1 h; live rebuild (spare VMs) a half-day
**References:** [node-bootstrap.md](node-bootstrap.md) (both rebuild paths, node-local state), backup-strategy.md "Layer 0"

The tabletop version is mandatory before any live attempt and already has
teeth — it verifies the **prerequisite chain**, which is where DR actually
fails:

1. **Access check (do this with the cluster assumed dead):** can you reach,
   without anything running in the cluster:
   - the B2 account console (login credentials — where do they live? If the
     answer is "Vaultwarden", that's **in the cluster**: confirm you have a
     working offline Vaultwarden export/client cache, or move the B2 login to
     a second store);
   - the `k3s-safeqbit-etcd` bucket key (root-only on the — dead — server
     nodes; the B2 console can mint a new one if you can log in);
   - the git repo (GitHub) + a machine with `kubectl`/`k3s`/`velero` CLIs;
   - the sealed-secrets private key (lives **only in etcd** → only reachable
     via the etcd snapshot restore path — this is why Layer 0 exists).
2. **Path walk-through:** narrate both node-bootstrap.md paths end-to-end
   (etcd-restore path vs git+key path), checking each referenced file,
   bucket, and command still exists. Note every step where you had to guess.
3. **Snapshot freshness:** `kubectl get etcdsnapshotfiles | grep s3://` shows
   entries newer than 7 days; pull one from B2 and check it's non-trivial in
   size (~29MB+).
4. Log gaps found → file them as improvement-plan items.

**Live variant (later):** 3 fresh VMs on Proxmox, restore per
node-bootstrap.md (`--cluster-reset-restore-path` from B2), point at git,
confirm Flux converges and one app (uptime-kuma) serves with data. Do not
touch the production VMs.

---

## Proposed rotation

| Quarter | Drill | Notes |
|---|---|---|
| Q3 2026 | A (Longhorn revert) + D tabletop | Both cheap; D-tabletop finds the access-chain gaps early |
| Q4 2026 | B1, then B2 (Velero passzilla) | The layer that has never moved data back from B2 |
| Q1 2027 | C1, then C2 (CNPG affine) | |
| Q2 2027 | D live (spare VMs) | Only after A–C pass and tabletop gaps are closed |

---

## Results log

| Date | Drill | Variant | Result | Time taken | Findings / follow-ups |
|---|---|---|---|---|---|
| _none yet_ | | | | | |
