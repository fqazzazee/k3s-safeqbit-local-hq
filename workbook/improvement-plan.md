# Cluster Improvement Plan

**Created:** 2026-05-27
**Cluster:** safeqbit-local-hq
**Status legend:** ☐ open · ◐ in-progress · ☑ done · ✗ wontfix

A prioritized backlog of fixes, improvements, and hardening items derived from the 2026-05-27 audit. Each item is self-contained: drop into it directly, do the fix, mark the box. Items group by priority but within a priority are independent unless a `Depends on:` line says otherwise.

---

## Priority key

- **P0 - Will recur soon, hits in production.** Likely to bite within the next month of normal operation. Tackle this week.
- **P1 - Single point of failure or hidden risk.** Currently fine but one event away from a real outage. Tackle this month.
- **P2 - Operational hygiene.** Won't break anything immediately; failing to do it compounds tech debt and creates worse cleanups later.
- **P3 - Nice to have / future state.** Worth tracking but not on the critical path.

---

## P0 - Will recur soon

### ☑ P0.1 Convert single-replica RWO-volume Deployments to `strategy: Recreate`

**Closed:** 2026-05-28 - audited by actual PVC access mode rather than presence. Added `strategy: Recreate` to 6 truly-RWO deployments: `affine/affine`, `immich/immich-server` (mixed RWO config + RWX data), `immich/immich-machine-learning`, `netbox/netbox-redis-tasks`, `passzilla/passzilla`, `monitoring/monitoring-grafana` (via Helm `deploymentStrategy`). Initially-flagged `authentik-server/worker`, `netbox/netbox`, `netbox/netbox-worker`, `photoprism/photoprism` were left on RollingUpdate - their PVCs are all RWX (NFS), no Multi-Attach risk.

**Why:** During Flux reconciles that change a pod template, `RollingUpdate` keeps the old pod alive until the new pod is Ready. But the new pod can't reach Ready because the RWO PVC is still attached to the old pod's node - Multi-Attach error, deadlock. We hit this on 2026-05-27 with `affine`, `passzilla`, `netbox-redis-tasks`, `immich-server`, `immich-machine-learning` and had to manually scale old ReplicaSets to 0.

**Current state - 11 deployments at risk:**
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

**Trade-off:** ~30s of downtime per rollout (old terminates fully before new starts). Acceptable for these workloads - none are user-facing HA-critical.

**Verification:** Trigger a re-reconcile (touch a label, e.g. `kubectl -n affine annotate deploy affine kubectl.kubernetes.io/restartedAt="$(date)"`) and confirm no Multi-Attach errors in events.

**Effort:** ~30 min. **Depends on:** none.

---

### ☑ P0.2 Scale CoreDNS to 3 replicas with hard host-spread

**Closed (for real): 2026-07-03.** The 2026-05-28 closure did not stick: the
coredns Deployment is owned by the k3s `Addon` controller whose applied state
includes `replicas: 1`, so the node reboots of late June silently reverted the
live-patch AND the node-local manifest edits. Final fix (PR #27): a Flux-managed
**cluster-proportional-autoscaler** (`configs/coredns-autoscaler.yaml`,
registry.k8s.io v1.10.3) targeting `deployment/coredns` — linear mode
`nodesPerReplica: 1, min: 2, max: 3` → 3 replicas, re-enforced every ~10s, so
any future k3s re-apply self-heals in seconds. Hard host-spread needed no work:
the bundled template already carries hostname `topologySpreadConstraints`
(DoNotSchedule) + required podAntiAffinity, so replicas land one per node.
This also finally satisfied the coredns PDB (`minAvailable: 2`, from P1.1) and
cleared the standing `KubePdbNotEnoughHealthyPods` alert.

> Lesson recorded: never "fix" a k3s-Addon-owned object by patching it or
> editing `/var/lib/rancher/k3s/server/manifests/` — both revert. Use a
> controller that continuously enforces, or `--disable` the addon and own it.

---

### ☑ P0.3 Fix passzilla session secret + pin image version

**Closed:** 2026-05-28 - pinned image `pglombardo/pwpush:latest` → `pglombardo/pwpush:2.7.0` (matches the version observed running). Added `envFrom: secretRef: name: passzilla-secret` to the deployment. Created `apps/.../passzilla/04-sealed-secret.yaml` as a stub with the placeholder `REPLACE_WITH_KUBESEAL_OUTPUT`. **Action required from you:** seal `/tmp/passzilla-secret-unsealed.yaml` (generated SECRET_KEY_BASE inside) with kubeseal and replace the stub file contents.

---

> **Follow-up 2026-07-03 (PR #17):** passzilla was OOMKilled every ~15 min for
> 22 days (1,671 restarts) — pwpush 2.7.0 (Rails 8 web + worker under foreman)
> idles at ~480Mi and the 512Mi limit left no headroom. Limit raised to 1Gi,
> request 128Mi→512Mi. Verified stable (>1h, 0 restarts, ~668Mi steady).

---

## P1 - Single point of failure

### ☐ P1.2 Ship etcd snapshots off-cluster

**Why:** k3s's default behavior is to take an etcd snapshot every 12 hours and keep 5, stored at `/var/lib/rancher/k3s/server/db/snapshots/` on each control-plane node. All snapshots are local. If the Proxmox host fire/corrupts/ransoms all 3 VMs simultaneously (shared storage, shared host, shared backup snapshot, shared malware), there's no off-cluster recovery point. Without etcd, the entire cluster config is gone - every Deployment, Service, Secret, PV must be reconstructed from the Flux repo, which doesn't include things like SealedSecret-decrypted state, in-cluster CRs etc.

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

Make this consistent with how the maintenance workbook documents node shutdown - etcd is the single most precious thing on the cluster.

**Verification:** After config change, `ls /var/lib/rancher/k3s/server/db/snapshots/ | wc -l` shows new snapshots with the new naming pattern; remote bucket lists them.

**Effort:** ~1.5 hr (S3 option). **Depends on:** decision on backend storage (could use existing B2 bucket since cap concerns are different for etcd-sized files).

---

### ☑ P1.1 PodDisruptionBudgets for critical control-plane / dataplane pods

**Closed:** 2026-05-28 - added PDBs for 6 multi-replica services. Cert-manager (controller + webhook + cainjector) configured via chart `podDisruptionBudget.enabled: true` values in `controllers/cert-manager.yaml`. Standalone PDB manifests in new file `configs/pdbs.yaml` for CoreDNS (`minAvailable: 2`), CNPG operator (`minAvailable: 1`), and nfs-subdir-external-provisioner (`minAvailable: 1`) - these charts don't expose a PDB value. After Flux reconcile, drain of any single control-plane node will succeed without violating PDB.

---

### ☑ P1.3 Bump single-instance CNPG clusters to `instances: 2`

**Closed:** 2026-05-28 - set `spec.instances: 2` on `affine-cnpg`, `netbox-cnpg`, and `grafana-cnpg`. CNPG will pg_basebackup the new standbys on next reconcile (takes a few minutes per cluster, no downtime on the existing primaries). Cost: 3 × 5 GiB extra PVCs ≈ 15 GiB cluster-wide after Longhorn replicas. Was well within available headroom post the orphan cleanup.

---

### ☑ P1.4 Wire up Alertmanager so firing alerts go somewhere

**Closed:** 2026-05-28 - enabled `alertmanager: enabled: true` in `controllers/monitoring.yaml` with 1Gi Longhorn storage, inline AM config routing to a Slack `slack-default` receiver. Webhook URL is read at runtime from `slack_api_url_file`, pointed at the mounted SealedSecret `alertmanager-slack-webhook` (stub in `configs/alertmanager-slack.yaml`). Suppressed the three k3s false-positives at the rule source via `defaultRules.rules.kubeProxy/Scheduler/ControllerManager: false` AND disabled the corresponding ServiceMonitors. Added `route.routes` to silence `Watchdog` and bump `severity=critical` alerts to 10s wait / 1h repeat. **Action required from you:** seal a Secret containing the Slack channel webhook URL into `alertmanager-slack.yaml`, set the correct channel name (`#k3s-alerts` placeholder), and re-commit. Until then, keep alertmanager disabled or AM pod will crash-loop.

> **Actually finished 2026-07-03** — the 05-28 wiring never delivered a single
> notification. Two stacked bugs, each masking the next (PRs #14, #15):
> (1) `secrets:` was nested under `alertmanager:` instead of
> `alertmanager.alertmanagerSpec:` — the chart silently ignored it and the
> webhook file was never mounted; (2) the `#k3s-alerts` channel override was a
> placeholder Slack couldn't resolve (404 channel_not_found) — removed; the
> webhook posts to its bound channel. Also added (PRs #16, #19, #20/#25/#26):
> ingresses + `externalUrl` for Alertmanager/Prometheus so notification links
> work (alertmanager/prometheus.local.safeqbit.com), a `VeleroBackupSucceeded`
> info route (one ✅ per successful backup), and a weekly Monday-08:00-ET
> backup digest CronJob (`configs/backup-summary-report.yaml`) with per-schedule
> status/next-run/retention + B2 bucket usage.

---

## P2 - Operational hygiene

### ☑ P2.2 Move Velero off Backblaze B2 (or reduce frequency)

> **Partial progress 2026-05-28:** the "reduce frequency" half is done - Velero schedules were fully rewritten under P2.1. Killed `daily-everything` and `weekly-everything`, replaced with per-workload bi-monthly schedules staggered so only one runs per day (5th/7th/9th/11th/13th/20th/22nd/24th/26th/28th), plus weekly for high-churn workloads (affine Sunday, netbox Wednesday). TTL uniform 180d. See [backup-strategy.md](backup-strategy.md). The "move off B2" half (R2 or NFS) is still open.
>
> **Closed 2026-07-03: staying on B2 deliberately.** Backblaze made all
> standard API calls free effective 2026-05-01, removing the transaction-cap
> rationale for moving (account caps persist for cardless accounts, but PR #22
> cut Velero's bucket polling 1m→1h ≈ 60x, well under them; R2 also requires a
> payment method, so it solves nothing we have). Full tuning same day
> (PRs #18, #29, #30): NFS volumes skipped (TrueNAS owns them), Prometheus TSDB
> excluded (accepted DR loss), CNPG replica PVCs skipped (halves DB churn),
> immich-bimonthly added (immich had NO backup at all — plain-Deployment
> postgres, no CNPG layer), per-workload TTLs (60d most / 90d authentik /
> 28d passzilla / 180d vaultwarden). Bucket at ~2.8GiB of 10GiB free tier;
> the weekly digest warns at 80%. Revisit only if that warning fires
> (then: MinIO on TrueNAS, not R2).



**Why:** Today the B2 free-tier transaction cap was exceeded by Velero's hourly kopia-maintain jobs across 6 workloads. We bandaged it to 168h, but the daily Velero backups themselves (`daily-everything` cron `0 3 * * *`) also write to B2 and will eventually hit the cap depending on data volume.

**Current state:**
- BSL `default`: B2 endpoint, currently `Available`
- 6 Velero `Schedule` resources with TTLs of 720h-4320h (good retention is set)
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
- **Cluster-wide rebuild from Flux + Velero** (simulated DR - what if every node is wiped? Spin up new 3-node cluster, point Flux at the repo, restore from Velero. Document gaps.)

Then run one of these per quarter, log results, refine docs.

**Effort:** ~4 hr (first drill including doc writing). **Depends on:** P2.2 if you're switching backends.

---

### ☑ P2.1 CNPG ScheduledBackup retention policy

**Closed:** 2026-05-28 - added `cnpg-backup-retention` CronJob in `cnpg-system` (file `configs/cnpg-backup-retention.yaml`). Runs daily at 05:00 UTC, lists CNPG Backup CRs per cluster, deletes oldest beyond retention via `kubectl delete`. Owner-ref cascades VolumeSnapshot + Longhorn snapshot cleanup. Per-cluster retention: authentik 10 (weekly cadence ≈ 10 weeks), affine/netbox/grafana 30 each (daily ≈ 1 month). Also added missing ScheduledBackups for `netbox-cnpg` (daily 02:15) and `grafana-cnpg` (daily 02:30); changed `authentik-cnpg` from daily to weekly Sunday 02:45; `affine-cnpg` stays daily (now 02:00). Full layout in [backup-strategy.md](backup-strategy.md).

> **Gap fixed 2026-07-03 (PR #20):** guacamole was never added to the prune
> list — its weekly CNPG backups grew unbounded since 06-05. Added
> `prune guacamole 10`.

---

### ☑ P2.4 Replace `:latest` image tags

**Closed:** 2026-05-29 - passzilla pinned to `pglombardo/pwpush:2.7.0` as part of P0.3. Photoprism pinned to `photoprism/photoprism:260523` - that's the YYMMDD-coded stable tag matching the currently-running digest (`sha256:ee3d15c…`). PhotoPrism uses date-coded immutable tags as their stable channel; future bumps mean picking a newer date. Both stragglers now have explicit, reproducible tags.

---

### ☑ P2.5 Add Longhorn / CNPG / Velero Prometheus alert rules

**Closed:** 2026-05-29 - discovered Longhorn and Velero metric endpoints existed but had no ServiceMonitor pointing Prometheus at them (CNPG was already covered by operator-managed PodMonitors). Added two files in `configs/`:

- **`monitoring-servicemonitors.yaml`** - ServiceMonitor for `longhorn-backend.longhorn-system:9500` (port `manager`) and `velero.velero:8085` (port `http-monitoring`). Both labeled `release: monitoring` to match Prometheus operator selectors.
- **`monitoring-alert-rules.yaml`** - PrometheusRule `safeqbit-storage-and-backup` with 8 alerts across three groups:

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

**Note:** label is `state=` (not `robustness=` as I'd written in the original plan); BSL unavailability has no native gauge metric in this Velero version - symptom is `velero_backup_failure_total` ticking up, so `VeleroBackupFailureRateHigh` covers it. CNPG `cnpg_backup_status` metric was researched but doesn't exist - Backup CR status isn't exporter-exposed; the cnpg-backup-retention CronJob's own success/fail surfacing is left as a future improvement.

---

### ☑ P2.6 Workload balance - fewer pods land on server-03

**Closed:** 2026-05-29 - investigation showed **no scheduling defect, no fix needed.** Findings on 2026-05-29:

- Pod counts: server-01=23, server-02=25, server-03=13
- **Longhorn replicas: 18/18/18 (perfectly balanced)** - original hypothesis (volume-affinity pulling new replicas to 01/02) was wrong.
- **Non-PVC pods balance fine**: 24/21/29 - server-03 actually leads. Scheduler isn't avoiding it.
- **PVC-consumer pods skew toward 01/02**: 9/12/5. This is volume *attachment* stickiness, not replica placement - when a Longhorn volume is attached to a node, the pod that uses it stays pinned to that node. New pods land wherever the scheduler picks; on restart they often re-land on the same node (volume already attached) instead of triggering detach/re-attach.
- Only one workload uses zone-based topology (coredns, `whenUnsatisfiable: ScheduleAnyway`) - doesn't enforce, so doesn't exclude.
- Resource allocatable differs (server-03 has less memory: 14.4 GiB vs 18.9/22.4) which is also why scheduler appropriately keeps load off it.

No taints, no excluding affinity, no zone enforcement. The "imbalance" is just Longhorn attachment stickiness + resource-aware scheduling doing what they should. Finding captured here only - no repo PR required. If load on server-03 ever becomes a concern, the lever is `topologySpreadConstraints` on the higher-traffic apps; currently not worth the churn.

---

## P3 - Future state

### ☐ P3.1 Migrate CNPG backups to plugin-based (Barman) on S3

End state of P2.1 - proper native retention, point-in-time recovery via WAL archiving, off-cluster backup target. Bigger lift; do it after P2.1 has bought time.

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
3. Move closed items to the bottom of their section (don't delete - preserves history)

For ad-hoc additions, just append to the relevant priority section. Renumber on a quiet day if it bugs you.

---

## Future considerations

Items reviewed and consciously **deferred** - recommended approach is captured so they're a clean drop-in when picked up. These are not on any near-term track; revisit when there's appetite, not pressure.

---

### ☐ P3.3 Cluster-wide image policy + SBOM

**Reviewed:** 2026-05-29 - deferred. No controller installed; recommendation below.

**Original intent:** GitOps repo enforces no `:latest`, no untagged, optionally signed images (cosign). Future-proof against supply-chain compromise and accidental floating-tag drift. Candidate tools: Kyverno or OPA Gatekeeper.

**Current state (2026-05-29):** No admission/policy engine is installed. P2.4 already removed the last `:latest` tags (passzilla → `2.7.0`, photoprism → `260523`), so the cluster is *currently* compliant with a "no `:latest`" rule - which makes this a good time to add a guardrail that prevents regression, not a cleanup.

**Recommended approach when picked up:**

1. **Engine: Kyverno.** YAML-native policies (no Rego), first-class with GitOps/Flux, lowest authoring/maintenance overhead for a homelab. Install as a Flux `HelmRelease` (`kyverno/kyverno` chart) in its own `kyverno` namespace; add the HelmRepository to `controllers/sources.yaml` and the release to `controllers/kustomization.yaml`. Note Kyverno installs a validating webhook - keep `failurePolicy: Ignore` initially so a Kyverno outage can't wedge the API server / Flux reconciles.

2. **Mode: Audit first, then Enforce.** Roll out every `ClusterPolicy` with `validationFailureAction: Audit`. Watch `PolicyReport`/`ClusterPolicyReport` (and the Kyverno Prometheus metrics, which the existing kube-prometheus-stack can scrape via a ServiceMonitor) for a couple of weeks. Flip to `Enforce` only once reports are clean - flipping to Enforce blind risks a future chart bump or reconcile getting rejected and stalling Flux.

3. **Rules, in priority order:**
   - **`disallow-latest-tag`** (do first) - require an explicit, non-`:latest`, non-empty tag on every container/initContainer image. This is the literal "no `:latest`" goal and the cluster already passes it.
   - **`restrict-registries`** (worth it) - allowlist trusted registries (`docker.io`, `ghcr.io`, `quay.io`, `registry.k8s.io`, plus the specific app registries in use). Cheap blast-radius reduction against typo-squat / unexpected sources.
   - **`require-image-digest`** (stretch / likely skip) - pinning by `@sha256:` is the strongest reproducibility but **noisy**: nearly all current manifests and Helm charts reference tags, not digests, so expect a flood of audit findings and ongoing churn on every upstream bump. Not worth it without an image-automation workflow (Flux ImageUpdateAutomation or Renovate writing digests).

4. **SBOM / cosign signature verification: treat as a separate follow-up, not part of this item's initial rollout.** Every workload here is a *third-party public image we don't build* (pwpush, photoprism, authentik, immich, etc.), so:
   - **SBOM generation** doesn't apply - we're not the publisher. The relevant analogue is *consuming* upstream SBOM/provenance attestations, which most of these publishers don't provide consistently.
   - **Signature verification** (Kyverno `verifyImages` + cosign) requires per-publisher trust config - a keyless identity (OIDC issuer + subject regex) or public key for *each* image source. High setup and ongoing-maintenance cost for marginal benefit on a single-tenant homelab. If desired later, scaffold it disabled/audit-only against the one or two images that actually publish keyless signatures, and grow from there.

**Effort:** ~2-3 hr for Kyverno + the two recommended Audit policies; signature verification is a multi-hour project on its own. **Depends on:** none (but do it while the cluster is already `:latest`-clean so Audit starts green).

---

### ☐ P3.5 GitOps drift / Flux reconciliation alerts

**Reviewed:** 2026-05-29 - deferred. Metric path fully scoped below; safe and low-effort whenever picked up.

**Original intent:** Notify when any Kustomization or HelmRelease is `Ready=False` for an extended window. Would have caught the velero / nfs-provisioner / sealed-secrets `nodeSelector` issue automatically instead of by manual inspection.

**Key finding (2026-05-29):** The old `gotk_reconcile_condition` metric **no longer exists** in our Flux (v2.8.6) - verified by curling `:8080/metrics` on `kustomize-controller` and `helm-controller`: only `gotk_reconcile_duration_seconds` and `gotk_token_*` are exposed. So any alert rule written against `gotk_reconcile_condition{status="False"}` (the pattern in most older blog posts / the legacy Flux docs) would silently never fire. Don't go down that path.

**Recommended approach when picked up - kube-state-metrics Custom Resource State (CRS):**

The modern Flux monitoring pattern exposes CR readiness through kube-state-metrics, emitting `gotk_resource_info{exported_namespace, name, ready="True|False", suspended, customresource_kind, ...} == 1` per Flux object. We already run a shared kube-state-metrics (`monitoring-kube-state-metrics`, from kube-prometheus-stack).

1. **Feed KSM the Flux CRS config via the existing HelmRelease** (`controllers/monitoring.yaml`), under the chart's `kube-state-metrics:` values key:
   - `rbac.extraRules` granting `list`/`watch` on the Flux API groups (`kustomize.toolkit.fluxcd.io`, `helm.toolkit.fluxcd.io`, `source.toolkit.fluxcd.io`, `notification.…`, `image.…`).
   - `customResourceState.enabled: true` + `customResourceState.config` with the per-GVK `gotk_resource_info` definitions (canonical config: fluxcd `flux2-monitoring-example`, file `monitoring/controllers/kube-prometheus-stack/kube-state-metrics-config.yaml` - already retrieved and validated against our cluster on 2026-05-29).
   - **CRITICAL caveat:** the upstream example also sets `collectors: []` and `extraArgs: ["--custom-resource-state-only=true"]` because it runs a *dedicated* KSM instance. **Do NOT copy those two lines** onto our shared instance - they would disable every standard `kube_*` metric and break the existing dashboards and most kube-prometheus-stack alerts. Add *only* `rbac.extraRules` + `customResourceState`.

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

3. **Routing is already done** - these flow through the Alertmanager → Slack `#k3s-alerts` receiver wired in P1.4. No notification plumbing needed.

This naturally covers Kustomizations *and* HelmReleases (and GitRepository/HelmRepository/HelmChart), so it would catch both the Flux-layer and Helm-layer failures the original note worried about.

**Effort:** ~45 min. **Risk:** low (additive metrics + one rule), provided the `collectors: []` / `custom-resource-state-only` trap is avoided. **Depends on:** P1.4 (Alertmanager → Slack - done).
