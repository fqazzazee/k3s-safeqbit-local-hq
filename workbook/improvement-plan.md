# Cluster Improvement Plan

**Created:** 2026-05-27
**Cluster:** safeqbit-local-hq
**Status legend:** ☐ open · ◐ in-progress · ☑ done · ✗ wontfix

A prioritized backlog of fixes, improvements, and hardening items derived from the 2026-05-27 audit. Each item is self-contained: drop into it directly, do the fix, mark the box. Items group by priority but within a priority are independent unless a `Depends on:` line says otherwise.

---

## Priority key

- **P0 — Will recur soon, hits in production.** Likely to bite within the next month of normal operation. Tackle this week.
- **P1 — Single point of failure or hidden risk.** Currently fine but one event away from a real outage. Tackle this month.
- **P2 — Operational hygiene.** Won't break anything immediately; failing to do it compounds tech debt and creates worse cleanups later.
- **P3 — Nice to have / future state.** Worth tracking but not on the critical path.

---

## P0 — Will recur soon

### ☑ P0.1 Convert single-replica RWO-volume Deployments to `strategy: Recreate`

**Closed:** 2026-05-28 — audited by actual PVC access mode rather than presence. Added `strategy: Recreate` to 6 truly-RWO deployments: `affine/affine`, `immich/immich-server` (mixed RWO config + RWX data), `immich/immich-machine-learning`, `netbox/netbox-redis-tasks`, `passzilla/passzilla`, `monitoring/monitoring-grafana` (via Helm `deploymentStrategy`). Initially-flagged `authentik-server/worker`, `netbox/netbox`, `netbox/netbox-worker`, `photoprism/photoprism` were left on RollingUpdate — their PVCs are all RWX (NFS), no Multi-Attach risk.

**Why:** During Flux reconciles that change a pod template, `RollingUpdate` keeps the old pod alive until the new pod is Ready. But the new pod can't reach Ready because the RWO PVC is still attached to the old pod's node — Multi-Attach error, deadlock. We hit this on 2026-05-27 with `affine`, `passzilla`, `netbox-redis-tasks`, `immich-server`, `immich-machine-learning` and had to manually scale old ReplicaSets to 0.

**Current state — 11 deployments at risk:**
```
affine/affine
authentik/authentik-server         # managed by authentik HelmRelease, controller chart value
authentik/authentik-worker         # same as above
immich/immich-machine-learning
immich/immich-server
monitoring/monitoring-grafana
netbox/netbox
netbox/netbox-redis-tasks
netbox/netbox-worker
passzilla/passzilla
photoprism/photoprism
```

Already correct: `immich/immich-postgres`, `vaultwarden`, `flux-system/source-controller`.

**Fix:** Add to each Deployment spec in the repo:
```yaml
spec:
  strategy:
    type: Recreate
```
For Helm-managed (authentik, monitoring-grafana): set via chart values. For plain Deployments: edit the YAML directly.

**Trade-off:** ~30s of downtime per rollout (old terminates fully before new starts). Acceptable for these workloads — none are user-facing HA-critical.

**Verification:** Trigger a re-reconcile (touch a label, e.g. `kubectl -n affine annotate deploy affine kubectl.kubernetes.io/restartedAt="$(date)"`) and confirm no Multi-Attach errors in events.

**Effort:** ~30 min. **Depends on:** none.

---

### ☑ P0.2 Scale CoreDNS to 3 replicas with hard host-spread

**Closed:** 2026-05-28 — CoreDNS in k3s is managed by an `Addon` CR (not a HelmChart) sourced from `/var/lib/rancher/k3s/server/manifests/coredns.yaml`. Approach: (1) live-patched the Deployment (`replicas: 3` + pod-anti-affinity on hostname), confirming 3 pods running on 3 separate nodes; (2) edited the source manifest file on all 3 control-plane nodes to inject `replicas: 3`. The file already had `topologySpreadConstraints` on `kubernetes.io/hostname` with `whenUnsatisfiable: DoNotSchedule`, so spread is enforced even without my anti-affinity. Now persistent across k3s restarts on any single node. **Caveat:** k3s upgrades may regenerate this file from the embedded template — re-check after each k3s version bump. See workbook section "CoreDNS HA". 

> **Future improvement (P3?):** Long-term, replace this file-edit pattern with `--disable=coredns` on the k3s server flag + a Flux-managed CoreDNS HelmRelease. Cleaner GitOps, no node-local files.

---

### ☐ P0.2 Scale CoreDNS to 3 replicas with hard host-spread

**Why:** Today CoreDNS has 1 replica on `k3s-server-01`. When that node reboots (we just did one), every pod cluster-wide loses DNS until the pod restarts on another node. Today's incident only spared us because the rebooted node was `server-02`, not the one CoreDNS lived on. Next time it could be the other way.

**Current state:**
```
kube-system/coredns deployment: replicas=1, affinity={}, all pods on k3s-server-01
```

**Fix:**
1. The k3s embedded CoreDNS is deployed via the auto-deploying manifest at `/var/lib/rancher/k3s/server/manifests/coredns.yaml`. To change replica count cluster-wide and survive k3s upgrades, use k3s's HelmChartConfig:
   - Create `infrastructure/safeqbit-local-hq/controllers/coredns-config.yaml`:
     ```yaml
     apiVersion: helm.cattle.io/v1
     kind: HelmChartConfig
     metadata:
       name: rke2-coredns          # k3s uses this chart name in 1.30+
       namespace: kube-system
     spec:
       valuesContent: |-
         replicaCount: 3
         affinity:
           podAntiAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
             - labelSelector:
                 matchExpressions:
                 - key: k8s-app
                   operator: In
                 - values: [kube-dns]
               topologyKey: kubernetes.io/hostname
     ```
   - **Note:** verify chart name with `kubectl -n kube-system get helmcharts` — may be `coredns` or `rke2-coredns` depending on k3s version
2. Or simpler interim fix: `kubectl -n kube-system scale deploy coredns --replicas=3` (will revert on next k3s restart that re-applies the manifest)

**Verification:** `kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide` shows 3 pods on 3 different nodes. From any app pod: `nslookup kubernetes.default` still works after killing each CoreDNS pod individually.

**Effort:** ~45 min. **Depends on:** none.

---

### ☑ P0.3 Fix passzilla session secret + pin image version

**Closed:** 2026-05-28 — pinned image `pglombardo/pwpush:latest` → `pglombardo/pwpush:2.7.0` (matches the version observed running). Added `envFrom: secretRef: name: passzilla-secret` to the deployment. Created `apps/.../passzilla/04-sealed-secret.yaml` as a stub with the placeholder `REPLACE_WITH_KUBESEAL_OUTPUT`. **Action required from you:** seal `/tmp/passzilla-secret-unsealed.yaml` (generated SECRET_KEY_BASE inside) with kubeseal and replace the stub file contents.

---

### ☐ P0.3 Fix passzilla session secret + pin image version

**Why:**
1. `passzilla/passzilla-config` ConfigMap has no `SECRET_KEY_BASE` — pwpush's startup log says `"SECRET_KEY_BASE not set; generating a random key for this boot. Users will need to login again after container restart."` Every restart invalidates all active sessions. Combined with frequent restarts (98 today before the probe-timeout fix), this is user-hostile.
2. Image is `pglombardo/pwpush:latest` — an upstream push can break passzilla silently during a Flux reconcile or pod restart.

**Fix:**
1. Generate a `SECRET_KEY_BASE`:
   ```bash
   docker run --rm pglombardo/pwpush bundle exec rails secret
   ```
2. Add a SealedSecret `passzilla/passzilla-secret` with key `SECRET_KEY_BASE`
3. Update `apps/safeqbit-local-hq/passzilla/03-deployment.yaml` to mount it via `envFrom`:
   ```yaml
   envFrom:
     - configMapRef:
         name: passzilla-config
     - secretRef:
         name: passzilla-secret      # new
   ```
4. Replace `image: pglombardo/pwpush:latest` with a specific tag (check Docker Hub for current stable — e.g. `pglombardo/pwpush:v2.7.0`)

**Verification:** Pod restarts no longer log the SECRET_KEY_BASE warning; sessions persist across pod restart.

**Effort:** ~30 min. **Depends on:** none.

---

## P1 — Single point of failure

### ☐ P1.2 Ship etcd snapshots off-cluster

**Why:** k3s's default behavior is to take an etcd snapshot every 12 hours and keep 5, stored at `/var/lib/rancher/k3s/server/db/snapshots/` on each control-plane node. All snapshots are local. If the Proxmox host fire/corrupts/ransoms all 3 VMs simultaneously (shared storage, shared host, shared backup snapshot, shared malware), there's no off-cluster recovery point. Without etcd, the entire cluster config is gone — every Deployment, Service, Secret, PV must be reconstructed from the Flux repo, which doesn't include things like SealedSecret-decrypted state, in-cluster CRs etc.

**Current state:**
```
/etc/rancher/k3s/config.yaml: not present (defaults in use)
Snapshots present locally on each server-0X
Off-cluster: nothing
```

**Fix options:**
1. **Best:** Configure k3s to upload snapshots to S3 (B2, R2, or even your NFS via a script). Edit `/etc/rancher/k3s/config.yaml` on each server node:
   ```yaml
   etcd-s3: true
   etcd-s3-endpoint: s3.us-east-005.backblazeb2.com   # or wherever
   etcd-s3-bucket: k3s-safeqbit-etcd
   etcd-s3-access-key: <key>
   etcd-s3-secret-key: <secret>
   etcd-snapshot-schedule-cron: "0 */6 * * *"          # every 6h
   etcd-snapshot-retention: 20
   ```
   Restart `k3s.service`. This persists across reboots.
2. **Simpler:** A nightly CronJob that `kubectl cp`s the latest snapshot from each control-plane node to TrueNAS via NFS PVC.
3. **Easiest:** ssh-based push from a Linux box you control.

Make this consistent with how the maintenance workbook documents node shutdown — etcd is the single most precious thing on the cluster.

**Verification:** After config change, `ls /var/lib/rancher/k3s/server/db/snapshots/ | wc -l` shows new snapshots with the new naming pattern; remote bucket lists them.

**Effort:** ~1.5 hr (S3 option). **Depends on:** decision on backend storage (could use existing B2 bucket since cap concerns are different for etcd-sized files).

---

### ☑ P1.1 PodDisruptionBudgets for critical control-plane / dataplane pods

**Closed:** 2026-05-28 — added PDBs for 6 multi-replica services. Cert-manager (controller + webhook + cainjector) configured via chart `podDisruptionBudget.enabled: true` values in `controllers/cert-manager.yaml`. Standalone PDB manifests in new file `configs/pdbs.yaml` for CoreDNS (`minAvailable: 2`), CNPG operator (`minAvailable: 1`), and nfs-subdir-external-provisioner (`minAvailable: 1`) — these charts don't expose a PDB value. After Flux reconcile, drain of any single control-plane node will succeed without violating PDB.

---

### ☐ P1.1 PodDisruptionBudgets for critical control-plane / dataplane pods

**Why:** During a node drain (for upgrades, hardware maintenance), kubelet kills pods on the draining node. Without a PDB, all replicas of a service on that node die simultaneously. CNPG primaries and Longhorn instance-managers already have auto-generated PDBs (good). But CoreDNS, longhorn-manager, sealed-secrets-controller, and most app deployments don't.

**Current state:**
- 13 PDBs exist, all auto-created by CNPG/Longhorn/cloudflared
- Missing for: CoreDNS, longhorn-manager (DaemonSet, but still), sealed-secrets-controller, ingress-nginx (DaemonSet — has redundancy by virtue of running on every node), all app Deployments

**Fix:** Add `PodDisruptionBudget` resources alongside each Deployment. For singletons (`replicas: 1`), use `minAvailable: 0` or skip (PDB on a singleton means the node can't be drained at all). For multi-replica services use `minAvailable: 1` or `maxUnavailable: 1`.

Concrete additions (only where multi-replica):
- `kube-system/coredns` (after P0.2): `minAvailable: 2` out of 3
- `cert-manager/cert-manager` (already 2 replicas): `minAvailable: 1`
- `cert-manager/cert-manager-webhook` (already 2 replicas): `minAvailable: 1`
- `cert-manager/cert-manager-cainjector` (already 2 replicas): `minAvailable: 1`
- `nfs-provisioner` (already 2 replicas): `minAvailable: 1`

**Verification:** `kubectl get pdb -A` shows new PDBs; `kubectl drain k3s-server-01 --ignore-daemonsets --dry-run=client` would now succeed without violating any PDB.

**Effort:** ~45 min. **Depends on:** P0.2 (coredns PDB needs >1 replica first).

---

### ☑ P1.3 Bump single-instance CNPG clusters to `instances: 2`

**Closed:** 2026-05-28 — set `spec.instances: 2` on `affine-cnpg`, `netbox-cnpg`, and `grafana-cnpg`. CNPG will pg_basebackup the new standbys on next reconcile (takes a few minutes per cluster, no downtime on the existing primaries). Cost: 3 × 5 GiB extra PVCs ≈ 15 GiB cluster-wide after Longhorn replicas. Was well within available headroom post the orphan cleanup.

---

### ☐ P1.3 Bump single-instance CNPG clusters to `instances: 2`

**Why:** When `authentik-cnpg-1` corrupted today, the primary `authentik-cnpg-2` kept serving — that's exactly the value of `instances: 2`. The other clusters have no such fallback:

**Current state:**
| Cluster | Instances | Storage |
|---|---|---|
| `authentik/authentik-cnpg` | 2 | 10 GiB ✓ |
| `affine/affine-cnpg` | 1 | 5 GiB |
| `monitoring/grafana-cnpg` | 1 | 5 GiB |
| `netbox/netbox-cnpg` | 1 | 5 GiB |

If any of those single-instance DBs has its volume corrupt the same way authentik's did, the app loses its database until you restore from a Velero snapshot (hours of work, recent data lost). 2 instances ≈ doubles disk and gives sub-minute recovery.

**Fix:** Edit `spec.instances: 1` → `2` in:
- `apps/safeqbit-local-hq/affine/03-cnpg-cluster.yaml`
- `apps/safeqbit-local-hq/netbox/03-cnpg-cluster.yaml`
- `infrastructure/safeqbit-local-hq/configs/grafana-cnpg.yaml`

**Cost:** +15 GiB Longhorn storage across the cluster (3 PVCs × 5GiB × 3-replica factor = ~45 GiB total → ~30 GiB increase). Well within current free space (server-02: 135 GiB available post-cleanup).

**Verification:** `kubectl get cluster -A` shows `INSTANCES: 2 READY: 2` for all four.

**Effort:** ~20 min. **Depends on:** none. **Note:** CNPG will pg_basebackup the standbys, takes a few minutes per cluster.

---

### ☑ P1.4 Wire up Alertmanager so firing alerts go somewhere

**Closed:** 2026-05-28 — enabled `alertmanager: enabled: true` in `controllers/monitoring.yaml` with 1Gi Longhorn storage, inline AM config routing to a Slack `slack-default` receiver. Webhook URL is read at runtime from `slack_api_url_file`, pointed at the mounted SealedSecret `alertmanager-slack-webhook` (stub in `configs/alertmanager-slack.yaml`). Suppressed the three k3s false-positives at the rule source via `defaultRules.rules.kubeProxy/Scheduler/ControllerManager: false` AND disabled the corresponding ServiceMonitors. Added `route.routes` to silence `Watchdog` and bump `severity=critical` alerts to 10s wait / 1h repeat. **Action required from you:** seal a Secret containing the Slack channel webhook URL into `alertmanager-slack.yaml`, set the correct channel name (`#k3s-alerts` placeholder), and re-commit. Until then, keep alertmanager disabled or AM pod will crash-loop.

---

### ☐ P1.4 Wire up Alertmanager so firing alerts go somewhere

**Why:** Prometheus has **6 alerts currently firing**, including 3 `critical` (`KubeProxyDown`, `KubeSchedulerDown`, `KubeControllerManagerDown` — all false positives from k3s embedding these components, but still). One real alert today is `KubeJobNotCompleted` (from the velero kopia failures). **Nobody gets notified.** The monitoring chart was deployed with `alertmanager.enabled: false`.

**Current state:**
- `monitoring-kube-prometheus-stack` deployed
- `alertmanager: enabled: false` in `infrastructure/safeqbit-local-hq/controllers/monitoring.yaml`
- 243 PrometheusRules loaded, but no destination

**Fix:**
1. Enable alertmanager in `monitoring.yaml`:
   ```yaml
   alertmanager:
     enabled: true
     alertmanagerSpec:
       storage:
         volumeClaimTemplate:
           spec:
             storageClassName: longhorn
             resources:
               requests:
                 storage: 1Gi
     config:
       route:
         receiver: 'discord'        # or email, slack, etc.
       receivers:
       - name: 'discord'
         webhook_configs:
         - url: 'https://discord.com/api/webhooks/...'    # via SealedSecret
   ```
2. **Suppress the k3s false positives** — silence `KubeProxyDown`, `KubeSchedulerDown`, `KubeControllerManagerDown`, `KubeControllerManagerDown` via Alertmanager `inhibit_rules`, or set `kubeProxy.enabled: false`, `kubeScheduler.enabled: false`, `kubeControllerManager.enabled: false` in the kube-prometheus-stack values (cleaner).
3. **Add Longhorn-specific alerts** (no built-in rule for `LonghornVolumeRobustness != healthy`). The Longhorn chart's `metrics.serviceMonitor.enabled: true` exposes the metrics; rules need to be authored. See https://longhorn.io/docs/1.11.2/monitoring/alert-rules-example/

**Verification:** Trigger a known alert (e.g. scale a deployment to 0) and confirm the notification lands in your channel.

**Effort:** ~2 hr (longest item — alert design is the bulk). **Depends on:** decision on notification channel.

---

## P2 — Operational hygiene

### ☐ P2.2 Move Velero off Backblaze B2 (or reduce frequency)

> **Partial progress 2026-05-28:** the "reduce frequency" half is done — Velero schedules were fully rewritten under P2.1. Killed `daily-everything` and `weekly-everything`, replaced with per-workload bi-monthly schedules staggered so only one runs per day (5th/7th/9th/11th/13th/20th/22nd/24th/26th/28th), plus weekly for high-churn workloads (affine Sunday, netbox Wednesday). TTL uniform 180d. See [backup-strategy.md](backup-strategy.md). The "move off B2" half (R2 or NFS) is still open.



**Why:** Today the B2 free-tier transaction cap was exceeded by Velero's hourly kopia-maintain jobs across 6 workloads. We bandaged it to 168h, but the daily Velero backups themselves (`daily-everything` cron `0 3 * * *`) also write to B2 and will eventually hit the cap depending on data volume.

**Current state:**
- BSL `default`: B2 endpoint, currently `Available`
- 6 Velero `Schedule` resources with TTLs of 720h–4320h (good retention is set)
- `defaultRepoMaintainFrequency: 168h0m0s` (post-fix)

**Fix options:**
1. **Cloudflare R2** (recommended): 10M Class A + 1M Class B free per month vs B2's 2500/day. ~400x more headroom. Switch BSL config:
   ```yaml
   config:
     region: auto
     s3ForcePathStyle: "true"
     s3Url: https://<account>.r2.cloudflarestorage.com
   ```
2. **Local NFS** (your TrueNAS at 10.10.10.5): Use a `BackupStorageLocation` of provider `aws` pointed at a local MinIO container, or use Velero's local filesystem provider. Pro: free, low latency. Con: not off-site, doesn't protect against site loss.
3. **Stay on B2 + lower frequency further**: stretch `daily-everything` to weekly, accept lower RPO.

**Verification:** Test restore from new BSL works end-to-end (restore a sample namespace).

**Effort:** ~3 hr (includes restore drill). **Depends on:** P2.3 (test restore should be part of any backend change).

---

### ☐ P2.3 Document and execute a quarterly restore drill

**Why:** Backups that haven't been tested aren't backups. We have multiple layers (CNPG ScheduledBackup → Longhorn snapshot, Velero → B2, etcd → local) but have never verified end-to-end recovery.

**Fix:** Document a restore drill in `workbook/restore-drills.md` covering:
- **Single-PVC restore from Longhorn snapshot** (e.g. recover one app's data without touching the rest)
- **Full namespace restore from Velero** (e.g. delete authentik namespace, restore from yesterday's Velero backup, verify login still works)
- **Single CNPG cluster restore** (e.g. drop database, restore from latest Backup CR)
- **Cluster-wide rebuild from Flux + Velero** (simulated DR — what if every node is wiped? Spin up new 3-node cluster, point Flux at the repo, restore from Velero. Document gaps.)

Then run one of these per quarter, log results, refine docs.

**Effort:** ~4 hr (first drill including doc writing). **Depends on:** P2.2 if you're switching backends.

---

### ☑ P2.1 CNPG ScheduledBackup retention policy

**Closed:** 2026-05-28 — added `cnpg-backup-retention` CronJob in `cnpg-system` (file `configs/cnpg-backup-retention.yaml`). Runs daily at 05:00 UTC, lists CNPG Backup CRs per cluster, deletes oldest beyond retention via `kubectl delete`. Owner-ref cascades VolumeSnapshot + Longhorn snapshot cleanup. Per-cluster retention: authentik 10 (weekly cadence ≈ 10 weeks), affine/netbox/grafana 30 each (daily ≈ 1 month). Also added missing ScheduledBackups for `netbox-cnpg` (daily 02:15) and `grafana-cnpg` (daily 02:30); changed `authentik-cnpg` from daily to weekly Sunday 02:45; `affine-cnpg` stays daily (now 02:00). Full layout in [backup-strategy.md](backup-strategy.md).

---

### ☐ P2.1 CNPG ScheduledBackup retention policy

**Why:** Today we manually deleted 119 old Backup CRs to get under the 250-snapshot-per-volume Longhorn cap. Without a retention policy, daily CNPG backups will accumulate indefinitely and hit the cap again. Even with the cron fix (daily instead of hourly), in ~6 months we'll be back at 180+ snapshots on the authentik volume.

**Current state:**
- `authentik/authentik-cnpg-backup`: daily at 02:00 UTC, no retention
- `affine/affine-cnpg-backup`: weekly on Sunday 03:00 UTC, no retention
- CNPG `ScheduledBackup` CRD doesn't have a built-in retention field

**Fix options:**
1. **Best:** Switch from `volumeSnapshot` method to CNPG's plugin-based backup (Barman) which has native retention. Requires moving backup target from Longhorn snapshots to S3-compatible storage. Bigger change.
2. **Pragmatic:** A weekly CronJob that runs:
   ```bash
   kubectl -n <ns> get backups.postgresql.cnpg.io --sort-by=.metadata.creationTimestamp \
     -o jsonpath='{.items[*].metadata.name}' \
     | tr ' ' '\n' | head -n -30 \
     | xargs -r kubectl -n <ns> delete backup.postgresql.cnpg.io
   ```
   Keeps newest 30 backups per cluster, deletes the rest. Owner-ref cascades VolumeSnapshot + Longhorn snapshot cleanup.
3. Author this as a CronJob per CNPG namespace, or one CronJob with a list.

**Verification:** A month after deployment, `kubectl get backups -A | wc -l` shows count plateauing at ~30 per cluster, not climbing.

**Effort:** ~1 hr. **Depends on:** none.

---

### ☑ P2.4 Replace `:latest` image tags

**Closed:** 2026-05-29 — passzilla pinned to `pglombardo/pwpush:2.7.0` as part of P0.3. Photoprism pinned to `photoprism/photoprism:260523` — that's the YYMMDD-coded stable tag matching the currently-running digest (`sha256:ee3d15c…`). PhotoPrism uses date-coded immutable tags as their stable channel; future bumps mean picking a newer date. Both stragglers now have explicit, reproducible tags.

---

### ☑ P2.5 Add Longhorn / CNPG / Velero Prometheus alert rules

**Closed:** 2026-05-29 — discovered Longhorn and Velero metric endpoints existed but had no ServiceMonitor pointing Prometheus at them (CNPG was already covered by operator-managed PodMonitors). Added two files in `configs/`:

- **`monitoring-servicemonitors.yaml`** — ServiceMonitor for `longhorn-backend.longhorn-system:9500` (port `manager`) and `velero.velero:8085` (port `http-monitoring`). Both labeled `release: monitoring` to match Prometheus operator selectors.
- **`monitoring-alert-rules.yaml`** — PrometheusRule `safeqbit-storage-and-backup` with 8 alerts across three groups:

| Group | Alert | Expression | Severity |
|---|---|---|---|
| longhorn | `LonghornVolumeDegraded` | `longhorn_volume_robustness{state="degraded"} == 1 for 10m` | warning |
| longhorn | `LonghornVolumeFaulted` | `longhorn_volume_robustness{state="faulted"} == 1 for 1m` | critical |
| longhorn | `LonghornVolumeSnapshotCountHigh` | `count by (volume) (longhorn_snapshot_actual_size_bytes) > 200 for 30m` | warning |
| longhorn | `LonghornNodeStorageLow` | `usage/capacity > 0.85 for 30m` | warning |
| cnpg | `CNPGReplicationLagHigh` | `cnpg_pg_replication_lag > 300 for 10m` | warning |
| cnpg | `CNPGExporterDown` | `cnpg_collector_up == 0 for 5m` | warning |
| velero | `VeleroBackupFailed` | `velero_backup_last_status{schedule!=""} == 0 for 5m` | warning |
| velero | `VeleroBackupFailureRateHigh` | `increase(velero_backup_failure_total[24h]) > 2 for 5m` | warning |

**Note:** label is `state=` (not `robustness=` as I'd written in the original plan); BSL unavailability has no native gauge metric in this Velero version — symptom is `velero_backup_failure_total` ticking up, so `VeleroBackupFailureRateHigh` covers it. CNPG `cnpg_backup_status` metric was researched but doesn't exist — Backup CR status isn't exporter-exposed; the cnpg-backup-retention CronJob's own success/fail surfacing is left as a future improvement.

---

### ☑ P2.6 Workload balance — fewer pods land on server-03

**Closed:** 2026-05-29 — investigation showed **no scheduling defect, no fix needed.** Findings on 2026-05-29:

- Pod counts: server-01=23, server-02=25, server-03=13
- **Longhorn replicas: 18/18/18 (perfectly balanced)** — original hypothesis (volume-affinity pulling new replicas to 01/02) was wrong.
- **Non-PVC pods balance fine**: 24/21/29 — server-03 actually leads. Scheduler isn't avoiding it.
- **PVC-consumer pods skew toward 01/02**: 9/12/5. This is volume *attachment* stickiness, not replica placement — when a Longhorn volume is attached to a node, the pod that uses it stays pinned to that node. New pods land wherever the scheduler picks; on restart they often re-land on the same node (volume already attached) instead of triggering detach/re-attach.
- Only one workload uses zone-based topology (coredns, `whenUnsatisfiable: ScheduleAnyway`) — doesn't enforce, so doesn't exclude.
- Resource allocatable differs (server-03 has less memory: 14.4 GiB vs 18.9/22.4) which is also why scheduler appropriately keeps load off it.

No taints, no excluding affinity, no zone enforcement. The "imbalance" is just Longhorn attachment stickiness + resource-aware scheduling doing what they should. Finding captured here only — no repo PR required. If load on server-03 ever becomes a concern, the lever is `topologySpreadConstraints` on the higher-traffic apps; currently not worth the churn.

---

## P3 — Future state

### ☐ P3.1 Migrate CNPG backups to plugin-based (Barman) on S3

End state of P2.1 — proper native retention, point-in-time recovery via WAL archiving, off-cluster backup target. Bigger lift; do it after P2.1 has bought time.

### ☐ P3.2 Network policies (deny-by-default within namespaces)

The cluster runs everything as default-allow. Kube-router netpol is already enforcing what's there; just no policies are authored. Worth adding minimal deny-all + per-app allow rules eventually for blast-radius reduction.

### ☐ P3.4 Document on-call playbooks

Per top-N alerts (degraded volume, CNPG failover, etcd quorum loss, ingress 5xx spike), write a one-page runbook each. Reference from the alert annotation `runbook_url`.

> P3.3 (image policy + SBOM) and P3.5 (Flux reconciliation alerts) were reviewed on 2026-05-29 and **deferred** rather than implemented. Full context + recommended approach moved to [Future considerations](#future-considerations) at the bottom of this page.

---

## Tracking

When picking an item up:
1. Change `☐` to `◐` and add `**Assigned:** <name> **Started:** YYYY-MM-DD` under the title
2. When done, change to `☑` and add `**Closed:** YYYY-MM-DD` with one-line of what was actually done
3. Move closed items to the bottom of their section (don't delete — preserves history)

For ad-hoc additions, just append to the relevant priority section. Renumber on a quiet day if it bugs you.

---

## Future considerations

Items reviewed and consciously **deferred** — recommended approach is captured so they're a clean drop-in when picked up. These are not on any near-term track; revisit when there's appetite, not pressure.

---

### ☐ P3.3 Cluster-wide image policy + SBOM

**Reviewed:** 2026-05-29 — deferred. No controller installed; recommendation below.

**Original intent:** GitOps repo enforces no `:latest`, no untagged, optionally signed images (cosign). Future-proof against supply-chain compromise and accidental floating-tag drift. Candidate tools: Kyverno or OPA Gatekeeper.

**Current state (2026-05-29):** No admission/policy engine is installed. P2.4 already removed the last `:latest` tags (passzilla → `2.7.0`, photoprism → `260523`), so the cluster is *currently* compliant with a "no `:latest`" rule — which makes this a good time to add a guardrail that prevents regression, not a cleanup.

**Recommended approach when picked up:**

1. **Engine: Kyverno.** YAML-native policies (no Rego), first-class with GitOps/Flux, lowest authoring/maintenance overhead for a homelab. Install as a Flux `HelmRelease` (`kyverno/kyverno` chart) in its own `kyverno` namespace; add the HelmRepository to `controllers/sources.yaml` and the release to `controllers/kustomization.yaml`. Note Kyverno installs a validating webhook — keep `failurePolicy: Ignore` initially so a Kyverno outage can't wedge the API server / Flux reconciles.

2. **Mode: Audit first, then Enforce.** Roll out every `ClusterPolicy` with `validationFailureAction: Audit`. Watch `PolicyReport`/`ClusterPolicyReport` (and the Kyverno Prometheus metrics, which the existing kube-prometheus-stack can scrape via a ServiceMonitor) for a couple of weeks. Flip to `Enforce` only once reports are clean — flipping to Enforce blind risks a future chart bump or reconcile getting rejected and stalling Flux.

3. **Rules, in priority order:**
   - **`disallow-latest-tag`** (do first) — require an explicit, non-`:latest`, non-empty tag on every container/initContainer image. This is the literal "no `:latest`" goal and the cluster already passes it.
   - **`restrict-registries`** (worth it) — allowlist trusted registries (`docker.io`, `ghcr.io`, `quay.io`, `registry.k8s.io`, plus the specific app registries in use). Cheap blast-radius reduction against typo-squat / unexpected sources.
   - **`require-image-digest`** (stretch / likely skip) — pinning by `@sha256:` is the strongest reproducibility but **noisy**: nearly all current manifests and Helm charts reference tags, not digests, so expect a flood of audit findings and ongoing churn on every upstream bump. Not worth it without an image-automation workflow (Flux ImageUpdateAutomation or Renovate writing digests).

4. **SBOM / cosign signature verification: treat as a separate follow-up, not part of this item's initial rollout.** Every workload here is a *third-party public image we don't build* (pwpush, photoprism, authentik, immich, etc.), so:
   - **SBOM generation** doesn't apply — we're not the publisher. The relevant analogue is *consuming* upstream SBOM/provenance attestations, which most of these publishers don't provide consistently.
   - **Signature verification** (Kyverno `verifyImages` + cosign) requires per-publisher trust config — a keyless identity (OIDC issuer + subject regex) or public key for *each* image source. High setup and ongoing-maintenance cost for marginal benefit on a single-tenant homelab. If desired later, scaffold it disabled/audit-only against the one or two images that actually publish keyless signatures, and grow from there.

**Effort:** ~2–3 hr for Kyverno + the two recommended Audit policies; signature verification is a multi-hour project on its own. **Depends on:** none (but do it while the cluster is already `:latest`-clean so Audit starts green).

---

### ☐ P3.5 GitOps drift / Flux reconciliation alerts

**Reviewed:** 2026-05-29 — deferred. Metric path fully scoped below; safe and low-effort whenever picked up.

**Original intent:** Notify when any Kustomization or HelmRelease is `Ready=False` for an extended window. Would have caught the velero / nfs-provisioner / sealed-secrets `nodeSelector` issue automatically instead of by manual inspection.

**Key finding (2026-05-29):** The old `gotk_reconcile_condition` metric **no longer exists** in our Flux (v2.8.6) — verified by curling `:8080/metrics` on `kustomize-controller` and `helm-controller`: only `gotk_reconcile_duration_seconds` and `gotk_token_*` are exposed. So any alert rule written against `gotk_reconcile_condition{status="False"}` (the pattern in most older blog posts / the legacy Flux docs) would silently never fire. Don't go down that path.

**Recommended approach when picked up — kube-state-metrics Custom Resource State (CRS):**

The modern Flux monitoring pattern exposes CR readiness through kube-state-metrics, emitting `gotk_resource_info{exported_namespace, name, ready="True|False", suspended, customresource_kind, ...} == 1` per Flux object. We already run a shared kube-state-metrics (`monitoring-kube-state-metrics`, from kube-prometheus-stack).

1. **Feed KSM the Flux CRS config via the existing HelmRelease** (`controllers/monitoring.yaml`), under the chart's `kube-state-metrics:` values key:
   - `rbac.extraRules` granting `list`/`watch` on the Flux API groups (`kustomize.toolkit.fluxcd.io`, `helm.toolkit.fluxcd.io`, `source.toolkit.fluxcd.io`, `notification.…`, `image.…`).
   - `customResourceState.enabled: true` + `customResourceState.config` with the per-GVK `gotk_resource_info` definitions (canonical config: fluxcd `flux2-monitoring-example`, file `monitoring/controllers/kube-prometheus-stack/kube-state-metrics-config.yaml` — already retrieved and validated against our cluster on 2026-05-29).
   - **CRITICAL caveat:** the upstream example also sets `collectors: []` and `extraArgs: ["--custom-resource-state-only=true"]` because it runs a *dedicated* KSM instance. **Do NOT copy those two lines** onto our shared instance — they would disable every standard `kube_*` metric and break the existing dashboards and most kube-prometheus-stack alerts. Add *only* `rbac.extraRules` + `customResourceState`.

2. **Add a `PrometheusRule`** (new `configs/monitoring-flux-rules.yaml`, or a `flux` group appended to `configs/monitoring-alert-rules.yaml`), labelled `release: monitoring` so the Prometheus `ruleSelector` picks it up:
   ```yaml
   - alert: FluxReconciliationFailure
     expr: |
       max by (customresource_kind, exported_namespace, name) (
         gotk_resource_info{ready="False", suspended!="true"}
       ) == 1
     for: 15m          # plan said >5m; 15m avoids flapping during legitimately long reconciles/installs
     labels:
       severity: warning
     annotations:
       summary: "Flux {{ $labels.customresource_kind }} {{ $labels.exported_namespace }}/{{ $labels.name }} not ready"
       description: "{{ $labels.customresource_kind }} {{ $labels.exported_namespace }}/{{ $labels.name }} has been Ready=False for >15m. Run `flux get all -A` / check the controller logs."
   ```
   Optionally a second `FluxSuspended` info-level alert on `gotk_resource_info{suspended="true"}` so long-lived manual suspends don't get forgotten.

3. **Routing is already done** — these flow through the Alertmanager → Slack `#k3s-alerts` receiver wired in P1.4. No notification plumbing needed.

This naturally covers Kustomizations *and* HelmReleases (and GitRepository/HelmRepository/HelmChart), so it would catch both the Flux-layer and Helm-layer failures the original note worried about.

**Effort:** ~45 min. **Risk:** low (additive metrics + one rule), provided the `collectors: []` / `custom-resource-state-only` trap is avoided. **Depends on:** P1.4 (Alertmanager → Slack — done).
