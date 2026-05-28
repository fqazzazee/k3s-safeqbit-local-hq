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

### ☐ P2.2 Move Velero off Backblaze B2 (or reduce frequency)

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

### ☐ P2.4 Replace `:latest` image tags

**Why:** Two app images in the repo use `:latest`:
- `apps/safeqbit-local-hq/passzilla/03-deployment.yaml`: `pglombardo/pwpush:latest`
- `apps/safeqbit-local-hq/photoprism/04-photoprism.yaml`: `photoprism/photoprism:latest`

Either of these doing a breaking upstream push will silently roll out on the next pod restart. Cloudflared and other images in the repo are already pinned — these are the stragglers.

**Fix:** Replace with specific tags. Check current upstream stable releases. Add to commit message what tag/digest was chosen so future-you knows what was last known-good.

Optional bonus: pin to a digest (`pglombardo/pwpush@sha256:abc...`) for full reproducibility — but tags are usually enough.

**Effort:** ~10 min per image. **Depends on:** P0.3 (which already addresses passzilla).

---

### ☐ P2.5 Add Longhorn / CNPG / Velero Prometheus alert rules

**Why:** The kube-prometheus-stack has 243 rules but **zero of them cover Longhorn, CNPG, or Velero** — exactly the subsystems where today's incidents originated. Without alerts, we only learn about degraded volumes / failed backups / snapshot count overruns when something downstream breaks.

**Concrete alerts to author (PrometheusRule CR in `monitoring` ns):**

| Alert | Expression | Severity |
|---|---|---|
| `LonghornVolumeDegraded` | `longhorn_volume_robustness{robustness="degraded"} == 1 for 10m` | warning |
| `LonghornVolumeSnapshotCountHigh` | `longhorn_snapshot_actual_size_bytes count by (volume) > 200` | warning |
| `LonghornNodeStorageLow` | `longhorn_node_storage_usage_bytes / longhorn_node_storage_capacity_bytes > 0.85` | warning |
| `CNPGBackupFailed` | `cnpg_backup_status{status="failed"} == 1` | warning |
| `CNPGReplicationLag` | `cnpg_pg_replication_lag > 300` | warning |
| `VeleroBackupFailed` | `velero_backup_failure_total > 0` | warning |
| `VeleroBSLUnavailable` | `velero_backup_storage_location_status{status="Unavailable"} == 1` | critical |

**Effort:** ~2 hr (look up exact metric names per chart, test each expression). **Depends on:** P1.4 (Alertmanager wired up).

---

### ☐ P2.6 Workload balance — fewer pods land on server-03

**Why:** Post-cleanup distribution is k3s-server-01: 23, server-02: 23, server-03: 9. Server-03 is significantly under-utilized. Cause is likely volume-aware scheduling pulling new pods toward nodes where Longhorn volumes already exist, plus server-03's zone label (`dumpling` vs others' `meatball`) potentially mattering somewhere.

**Investigation needed:**
- Are app PVCs disproportionately replica-located on servers 01/02 vs 03? `kubectl -n longhorn-system get replicas.longhorn.io | awk '{print $4}' | sort | uniq -c`
- Are any deployments using `topology.kubernetes.io/zone` as a topologyKey? (We didn't see that in the affinity audit, but worth checking.)

**Fix:** Probably none required — distribution will rebalance organically as pods restart over time. But worth verifying server-03 isn't accidentally excluded from something. Could add `topologySpreadConstraints` with `topology.kubernetes.io/zone` to specific apps if we want zone-aware spreading.

**Effort:** ~30 min investigation, maybe ~30 min fix if any found.

---

## P3 — Future state

### ☐ P3.1 Migrate CNPG backups to plugin-based (Barman) on S3

End state of P2.1 — proper native retention, point-in-time recovery via WAL archiving, off-cluster backup target. Bigger lift; do it after P2.1 has bought time.

### ☐ P3.2 Network policies (deny-by-default within namespaces)

The cluster runs everything as default-allow. Kube-router netpol is already enforcing what's there; just no policies are authored. Worth adding minimal deny-all + per-app allow rules eventually for blast-radius reduction.

### ☐ P3.3 Cluster-wide image policy + SBOM

GitOps repo could enforce: no `:latest`, no untagged, all images signed (cosign). Future-proof against supply-chain compromise. Tools: kyverno or OPA Gatekeeper.

### ☐ P3.4 Document on-call playbooks

Per top-N alerts (degraded volume, CNPG failover, etcd quorum loss, ingress 5xx spike), write a one-page runbook each. Reference from the alert annotation `runbook_url`.

### ☐ P3.5 GitOps drift / Flux reconciliation alerts

Notify when any Kustomization or HelmRelease is `Ready=False` for >5m. Today this would have caught the velero/nfs-provisioner/sealed-secrets `nodeSelector` issue without us noticing manually.

---

## Tracking

When picking an item up:
1. Change `☐` to `◐` and add `**Assigned:** <name> **Started:** YYYY-MM-DD` under the title
2. When done, change to `☑` and add `**Closed:** YYYY-MM-DD` with one-line of what was actually done
3. Move closed items to the bottom of their section (don't delete — preserves history)

For ad-hoc additions, just append to the relevant priority section. Renumber on a quiet day if it bugs you.
