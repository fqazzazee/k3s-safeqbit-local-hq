# Pulse

Single-pane-of-glass monitoring for the `safeqbit-local-hq` estate: the
**Proxmox hosts**, this **k3s cluster**, and the standalone **Docker host** — all
in one dashboard. ([rcourtman/Pulse](https://github.com/rcourtman/Pulse))

Server added 2026-06-29. k3s agent added 2026-06-30.

- **Namespace:** `pulse`
- **Hostname:** https://pulse.local.safeqbit.com (admin UI, internal only)
- **Manifests:** `apps/safeqbit-local-hq/pulse/`
- **Image:** `rcourtman/pulse:v6.4.1` (pinned; bump deliberately after reading upstream release notes — see [Version history](#version-history))
- **Server port:** `7655` (ClusterIP `pulse`, in-cluster DNS `pulse.pulse.svc.cluster.local:7655`)
- **Storage:** `pulse-data` 4Gi Longhorn RWO at `/data` — holds config, the **encrypted target credentials** (Proxmox tokens, agent tokens), discovered nodes, alert config, and history. **The only home for that config** (targets are added in the UI, not in Git).
- **Web-UI auth:** built-in, `PULSE_AUTH_USER` / `PULSE_AUTH_PASS` from SealedSecret `pulse-auth` (`03-sealed-secret.yaml`). Plaintext pass auto-hashed on startup → auth enforced from first boot, no open window. Admin password stored in Vaultwarden.
- **Backup:** `infrastructure/.../velero-schedule-pulse.yaml` — weekly to B2, Sundays 05:00 UTC, 60d retention (tuned down from 180d, 2026-07-03).

---

## HA model — why one replica, not many

Pulse is an **active poller with a single local data dir**, so it cannot run
active-active: two replicas would double-poll every target and race on `/data`
(and the RWO Longhorn volume can't multi-attach anyway). So the server runs
**exactly one replica** with `strategy: Recreate`.

Node-failure resilience comes from the storage layer instead, same pattern as
[uptime-kuma](uptime-kuma.md): the `pulse-data` volume is **Longhorn-replicated
across all 3 nodes**, and if the node holding the pod dies, Kubernetes reschedules
it on a survivor and Longhorn re-attaches the replicated data — monitoring resumes
in a few minutes, zero data loss. This was the explicit trade-off when deploying
(multi-replica was requested but isn't possible with Pulse's architecture).

---

## The k3s agent (single-replica Deployment)

To make the cluster appear in Pulse, the **unified agent** runs as a
**single-replica Deployment** (`06-agent-rbac.yaml`, `07-agent-sealed-secret.yaml`,
`08-agent-deployment.yaml`).

> **Why one replica, not a DaemonSet (hard-won):** it was *first* shipped as a
> DaemonSet (one pod per node) per the upstream example, and it **flapped** the
> agent online/offline on the dashboard. The Kubernetes module reads the *whole
> cluster* from the in-cluster API, so all 3 pods reported the identical view
> under the same `PULSE_AGENT_ID` and the server's single agent record raced
> between them. A DaemonSet is only correct when each node contributes its *own
> host metrics* (which we have off) — one reporter already sees everything.
> Fixed in PR `ops/pulse-agent-single-reporter`. `strategy: Recreate` so a
> rollout never briefly runs two reporters and re-triggers the flap.

- **Scope: Kubernetes-only.** `--enable-kubernetes`, `PULSE_ENABLE_HOST=false`.
  Reports node / pod / deployment health via the in-cluster API. Runs
  **non-privileged** (read-only root FS, `allowPrivilegeEscalation: false`, **no
  host mounts**).
- **Why no host metrics:** the nodes are Proxmox VMs (no real temps/SMART) and
  in-guest CPU/mem/disk is already covered by the Prometheus/Grafana stack. To add
  it later: set `PULSE_ENABLE_HOST=true`, add the `/proc` `/sys` `/` hostPath
  mounts + `privileged: true`, and reissue the token with `host-agent:report`
  (see upstream `docs/KUBERNETES.md`).
- **RBAC:** `pulse-agent` ServiceAccount + read-only ClusterRole (`get`/`list`/
  `watch` on nodes, pods, deployments) + binding. Monitoring read path only.
- **Token:** sealed `PULSE_TOKEN` (`pulse-agent-token`), generated in the Pulse UI
  with scope **`kubernetes:report`**.
- **`PULSE_AGENT_ID=safeqbit-local-hq`:** stable identity for the cluster's single
  reporter. (Also the lever that matters if this is ever scaled out: multiple
  pods sharing one ID is exactly what caused the dashboard flap — see the box
  above — so keep it to one reporter for the Kubernetes module.)

---

## Which targets need an agent?

| Target | Agent? | How |
|---|---|---|
| **Proxmox hosts** | No (agentless) | Add each PVE host in the UI with a Proxmox **API token**. Optional: install the host agent on a PVE node for temps/SMART. |
| **k3s cluster** | Yes — single Deployment | One agent reads the whole cluster via the in-cluster API (above). *Not* a DaemonSet — that flaps the dashboard. |
| **Docker host** (standalone) | Yes — one agent | Install the unified agent on that box (docker mode); it auto-detects Docker. Generate a token with the docker/host report scope in the UI. |

---

## Deploy

GitOps via Flux like every other app — no manual `kubectl apply`. The server went
straight to `main`; the agent went via PR (`ops/pulse-k3s-agent`, PR #9).

```sh
# Watch Flux pick it up
kubectl -n pulse get pods,pvc,ingress
kubectl -n pulse get certificate pulse-tls          # DNS-01 via Cloudflare — wait READY=True
kubectl -n pulse get deploy pulse-agent             # 1/1 ready (single reporter)
kubectl -n pulse logs deploy/pulse-agent --tail=20
```

## First-run / configuration (UI)

1. Log in at https://pulse.local.safeqbit.com as `admin` (password in Vaultwarden).
2. **Settings → API Tokens** — the `kubernetes:report` token for the DaemonSet
   already exists (sealed in Git). Generate additional tokens here for the Docker
   host agent if/when added.
3. **Add Proxmox hosts:** Settings → add each PVE node with a Proxmox API token.
4. Confirm the k3s cluster shows up as agent **`safeqbit-local-hq`**.

Target config lives only in the `pulse-data` PVC → it's covered by the weekly
Velero backup. Nothing about targets is in Git.

Two UI-only settings are **load-bearing** and must survive any reconfigure or
restore. See [the 2026-09-23 CPU burn](#the-2026-09-23-cpu-burn--rolling-cpu-evaluation-window):

- **Alerts → CPU evaluation window = "Current value"** (`alerts.json`:
  `metricEvaluationWindows: {all: {cpu: 0}}`). The 5-minute rolling window costs
  ~3 cores on v6.4.1. Set it to `0` explicitly; if the key is missing, Pulse
  falls back to the 300s default.
- **The Discord webhook's custom template escapes every value**:
  `{{.Message | jsonString}}`, and the same for `.ResourceName`, `.Node` and
  `.Level`. Without the escape, any alert text containing a `"` renders invalid
  JSON and dead-letters.

---

## Version history

Server and agent run the **same tag** — keep them in step. The four v6 manifest
requirements below apply to every v6 tag, so re-verify them (`--help` in a
throwaway pod) before any future bump rather than assuming they survived.

| Date | Tag | Scope | Notes |
| --- | --- | --- | --- |
| 2026-06-29 | `v5.1.35` | server | first deploy |
| 2026-06-30 | `v5.1.35` | + agent | k3s agent added |
| 2026-07-08 | `v5.1.36` | both | patch bump |
| 2026-07-25 | `v6.1.1` | both | major — one-way `/data` migration, see below |
| 2026-08-02 | `v6.1.2` | **server only** | patch, edited straight on GitHub; left the agent on `v6.1.1` and this doc unamended |
| 2026-08-18 | `v6.2.1` | both | minor, realigns the two on one tag |
| 2026-08-29 | `v6.4.1` | both | two minors + a patch; **went straight to `v6.4.1`** — `v6.4.0` shipped a non-executable embedded agent |

### v6.2.1 → v6.4.1 (2026-08-29)

Two minor releases and a patch, no migration and no manifest changes.
`v6.3.0` adds durable Patrol objectives and an Actions approval inbox;
`v6.4.0` rebuilds alert state from a durable event log (so incidents,
acknowledgements and snoozes survive a restart instead of flashing a false
all-clear), adds rolling-CPU and predictive-storage alerts, and routes
notification destinations by severity. Both say existing configuration stays
valid and the alert-identity migration runs automatically.

**Skip `v6.4.0`.** Its server image shipped an embedded Unified Agent that was
not executable, which breaks exactly the entrypoint this repo uses
(`command: /opt/pulse/bin/pulse-agent-linux-amd64`). `v6.4.1` is the fix.

All four v6 pins re-verified against the `v6.4.1` image before merging, using
the manifest's own binary path rather than the one on `$PATH`:
`--health-addr` still defaults to `127.0.0.1:9191` (so the explicit `:9191`
override is still load-bearing), `--disable-auto-update` and
`--enable-kubernetes` unchanged, and `/var/lib/pulse-agent` still the identity
path. RBAC unchanged — no new Kubernetes resource kinds in either release.

### v6.1.2 → v6.2.1 (2026-08-18)

Minor, no migration and no manifest changes. `v6.2.0` is the feature release
(external probes with signed config delivery, libvirt/XCP-ng/certificate
monitoring, ZFS-dataset / PBS-disk / LXC-filesystem coverage, an Actions inbox
with approve-execute-verify); `v6.2.1` is a hotfix on top of it (agent downloads
now follow redirects while still validating checksums, Agent Doctor credential
recovery, Pro activation, update-panel cache labels). Nothing in either touches
the Kubernetes agent path.

All four v6 pins re-verified against the `v6.2.1` binary before merging
(`pulse-agent --help` in a throwaway pod): `--health-addr` still defaults to
`127.0.0.1:9191`, `--disable-auto-update` and `--enable-kubernetes` unchanged,
`/var/lib/pulse-agent` still the identity path. RBAC unchanged — no new
Kubernetes resource kinds in the release notes; if a panel comes up empty, grep
the agent log for `kubernetes access forbidden (RBAC)`.

**Cutover, 2026-08-19 03:05 UTC — clean.** Both pods rolled on the first try,
`0 restarts`, no `error`-level lines on the server, no `kubernetes access
forbidden (RBAC)` on the agent. `/api/version` reports `6.2.1` with
`updateAvailable:false`; the agent logs `auto_update:false` and
`Health/metrics server listening addr :9191`, so the two pins that would fail
silently are both confirmed live. **No restore points were taken** — deliberate
call, given no `/data` migration in a minor. Flux needed a manual
`reconcile.fluxcd.io/requestedAt` nudge: `apps` had just failed a reconcile on
`dependency 'infrastructure-configs' is not ready` and would otherwise have sat
on the old revision for the rest of its 10m interval.

Two readings that look like failures and are not, both already documented for
v6.1.1 and both reproduced here: the agent logs one burst of `connection
refused` at startup when both pods roll together (it buffers and recovers on the
next 30s cycle), and `pulse_agent_destination_delivery_up` reads **0** until the
first cycle lands — measured at t+25s it was 0, at t+35s it was 1. Read it after
a full interval or you will chase a non-problem.

For a future *major*, still take the restore points and use the Flux
suspend/resume order from the v6 runbook below.

---

## v5 → v6 upgrade — DONE 2026-07-25

Cutover took ~3 minutes, both pods clean on the first roll, **0 restarts**. The
`/data` migration logged exactly one line — `Migrated legacy API tokens to
default organization binding` (6 tokens) — and nothing else. All targets came
back online: 3 Proxmox nodes, pbs02, the Docker host agent, and `in-cluster`.

Order that worked, worth repeating for the next major: Flux `apps` was still on
the pre-merge revision, so it got **suspended first**
(`kubectl patch kustomization apps -n flux-system --type merge -p
'{"spec":{"suspend":true}}'`), then both restore points were taken, then
`suspend:false` + a `reconcile.fluxcd.io/requestedAt` annotation to fire it
immediately. Suspending buys the window; without it a 10m interval can start the
one-way migration mid-backup.

Two v6 behaviours to expect and not panic about:

- **The agent logs one `connection refused` at startup** when both pods roll
  together — it buffers and recovers on the next 30s cycle. Read
  `pulse_agent_destination_delivery_up` *after* a full interval, not at t+30s.
- **`kubectl top` shows the server at ~87Mi** right after boot; it climbs as
  SQLite warms. The limit is sized for the warm figure, not this — and since
  2026-09-21 that limit is 2Gi, see [the OOM loop](#the-2026-09-21-oom-loop--unbounded-alert-event-store).

Restore points kept: `pulse-pre-v6-20260725` (rollback) and
`pulse-post-v6-20260725` (current), both as Velero/B2 backups *and* Longhorn
snapshots. The three v5-era weeklies and a duplicate pre-v6 backup were deleted
2026-07-25. Volume `actualSize` went 603MB → 884MB from holding the two
snapshots; that unwinds when they're trimmed.

Pre-existing, NOT caused by the upgrade: **pbs01 offline** (`no route to host`
to 10.10.10.20:8007, sits in the scheduler dead-letter queue) and the
`vm-prod-paw-conspaw01` disk-at-95% alerts.

Still on v5: the **Docker host and Proxmox node agents** (v5.1.35 / v5.1.36).
The server logs `Deprecated Unified Agent compatibility alias used; upgrade
agents to canonical /api/agents/agent/* endpoints` for them — they work, but
that's the next upgrade to schedule.

### Upgrade notes (kept for the next major)

`v5.1.36` → `v6.1.1`, direct — no intermediate hop. v5 went maintenance-only on
2026-07-04 (critical fixes only, through 2026-10-02) and `v5.1.36` is its final
release. Licensing is a non-issue: v6 dropped system-count metering and
Proxmox/Docker/Kubernetes monitoring, alerts, notifications and OIDC are all
Community — no activation, no outbound license call. Relay/Pro only gate remote
access, push notifications and extended history (14d/90d).

**The first v6 boot migrates `/data` in place, one way.** Re-pinning the old tag
does *not* undo it — a rollback means restoring the volume. Take the restore
points below *before* Flux reconciles the bump.

### Four things v6 breaks that a bare tag bump would not fix

Verified against the `v6.1.1` binaries before merging (throwaway pods, bogus
token, nothing registered server-side):

1. **`--health-addr` default moved from `:9191` to `127.0.0.1:9191`.** Confirmed
   in `/proc/net/tcp`: `0100007F:23E7`. The kubelet dials the *pod IP*, so both
   `tcpSocket: 9191` probes fail and the agent CrashLoopBackOffs. Pinned back to
   `:9191` in `08-agent-deployment.yaml`.
2. **Agent auto-update is on by default** (`"auto_update":true` at startup) — it
   updates ~5s after start, then hourly, which silently drifts the running binary
   off the pinned tag. `--disable-auto-update`.
3. **Agent persists its identity to `/var/lib/pulse-agent/agent-id`**, which the
   read-only root FS refuses. `emptyDir` mounted there (identity itself stays
   pinned by `PULSE_AGENT_ID`).
4. **RBAC was too narrow.** v6's Kubernetes module also reads namespaces,
   statefulsets/daemonsets/replicasets, jobs/cronjobs, PVCs, events,
   `metrics.k8s.io`, plus VolumeSnapshots and Velero backups for the
   Recovery/Protection view. Missing grants don't fail loudly — the agent logs
   `kubernetes access forbidden (RBAC)` and the panel renders empty.

Also in the same change: server memory limit `512Mi → 1Gi` (v5 already sat at
~392Mi and v6 adds Patrol + action/audit stores), `PULSE_TELEMETRY=false` (v6
pings `license.pulserelay.pro` on start and every 24h by default), and
`PULSE_PUBLIC_URL`. `FRONTEND_PORT` was already correct — v6 only honours the
legacy `PORT` as a deprecated fallback.

### Restore points (do this before merging)

```sh
# Layer 1 — Longhorn snapshot: fast local revert, seconds to take
VOL=$(kubectl -n pulse get pvc pulse-data -o jsonpath='{.spec.volumeName}')
kubectl -n longhorn-system get volume "$VOL"          # confirm attached/healthy

# Layer 3 — off-cluster copy to B2 (bare `kubectl get backup` in the velero ns
# resolves to backups.longhorn.io — always spell out backups.velero.io)
kubectl -n velero exec deploy/velero -- \
  /velero backup create pulse-pre-v6-$(date +%Y%m%d) --from-schedule pulse-weekly
kubectl -n velero get backups.velero.io | grep pulse-pre-v6

# Layer 0 — Pulse's own encrypted backup, UI: Settings → System → Recovery →
# Create Backup, then download it OFF the cluster. This is the only artefact
# that survives a bad /data migration without a volume restore.
```

Also worth a look before the bump: **Settings → System → Updates** in v5 renders
an upgrade plan that validates the server update path, agent continuity and token
scope.

### Post-upgrade checks

```sh
kubectl -n pulse rollout status deploy/pulse --timeout=5m
kubectl -n pulse rollout status deploy/pulse-agent --timeout=5m
kubectl -n pulse logs deploy/pulse-agent --tail=30 | grep -iE 'forbidden|health|auto-update'
kubectl -n pulse exec deploy/pulse -- wget -qO- 127.0.0.1:7655/api/version
# Anything past /api/version needs a session — port-forward and log in instead:
#   kubectl -n pulse port-forward deploy/pulse 17655:7655 &
#   curl -sc jar -X POST localhost:17655/api/login -H 'Content-Type: application/json' \
#     --data-raw '{"username":"admin","password":"<vaultwarden>"}'
#   curl -sb jar localhost:17655/api/monitoring/scheduler/health | jq
# NOTE: use 127.0.0.1 or the short name `pulse:7655` inside the pods. The image is
# Alpine/musl and the pod resolver runs ndots:5, so busybox wget hard-fails on the
# 4-dot pulse.pulse.svc.cluster.local ("bad address"). The Go agent is unaffected —
# it falls back to the absolute name. See project-dns-search-amplification.
```

Then in the UI: every Proxmox node still polling, the Docker host agent still
reporting, cluster `safeqbit-local-hq` online, and fire one test notification.
**Prune the old restore points only after the new version has soaked** — see
"Retiring pre-upgrade restore points" below.

### Retiring pre-upgrade restore points

Once v6 has run clean for a week (and the next scheduled `pulse-weekly` has
succeeded *on v6*), the pre-upgrade artefacts are dead weight:

```sh
# Velero: NEVER kubectl delete the CR — backup-sync resurrects it from B2
kubectl -n velero exec deploy/velero -- \
  /velero backup delete pulse-pre-v6-<DATE> --confirm

# Longhorn: delete the manual pre-upgrade snapshot in the UI (Volume → Snapshots),
# then let the weekly-trim RecurringJob reclaim the space
```

Keep the downloaded in-app encrypted backup until the *next* major upgrade — it's
small and it's the only version-portable copy of the target credentials.

---

## The 2026-09-21 OOM loop — unbounded alert event store

**Root cause: `/data/alerts/events.db` grew to 548MiB of live rows and the
alert engine could not process it without allocating unboundedly.** Recovered
by renaming the event store aside; Pulse recreates an empty one at startup.

### Why it grew

`alerts.json` carries **`maxAlertAgeDays: 0`, `maxAcknowledgedAgeDays: 0`,
`autoAcknowledgeAfterHours: 0`** — all three mean *never age out*. Alert events
had accumulated since the store was introduced. The SQLite header said 140,370
pages with a **freelist of 2**, so it was live data, not vacuum-able bloat.

### What tipped it

An alert/notification save in the UI at **2026-09-21 20:34:54 UTC**
(`alerts.json` + `apprise.enc`, same second) forced the alert engine to re-read
that history. Four minutes later the pod pinned its memory limit and never
served again — 37 restarts before it was caught.

### The signature, and how to recognise it again

- Memory climbs at a **constant ~9Mi/s from a cold start** and never plateaus.
- **Time-to-OOM scales linearly with the limit** — 3m46s at 1Gi, 7-9min at 2Gi.
  That ratio is the tell: a cache-pressure problem does not behave that way, a
  runaway allocation does.
- CPU sits ~2.5 cores while climbing, then **explodes to 6-9.5 cores** once
  memory pins the ceiling. That spike is the **Go GC death spiral**, not work —
  the runtime collecting against a heap it cannot shrink. It also starves the
  HTTP server, which is why readiness times out while the process is alive.
- The event store is **not being written** during the climb (`events.db` and
  its WAL are byte-identical) — it is being read. No SSH or process leak either.
- It is **node-independent**. Tested on server-01 and server-03: identical curve.

### Recovery

```sh
# 1. stop Flux reverting the scale-down (apps reconciles every 10m)
kubectl patch kustomization -n flux-system apps --type=merge -p '{"spec":{"suspend":true}}'
kubectl scale deploy -n pulse pulse --replicas=0

# 2. mount the PVC on its own (RWO — the server must be down first)
#    busybox pod with claimName: pulse-data at /data, then:
cd /data/alerts && for f in events.db events.db-wal events.db-shm; do
  mv "$f" "$f.incident-$(date +%Y%m%d).bak"; done

# 3. back up, and let Flux take over again
kubectl scale deploy -n pulse pulse --replicas=1
kubectl patch kustomization -n flux-system apps --type=merge -p '{"spec":{"suspend":false}}'
```

Everything else on the volume survives — targets, encrypted credentials,
`metrics.db`, and the alert *config*. Only alert event **history** is lost, and
the `.bak` files keep it on the PVC for forensics. Delete them once satisfied;
they are ~548MiB and Velero will otherwise keep backing them up. (This
incident's `.bak` files were deleted on 2026-09-23.)

**Set a non-zero `maxAlertAgeDays` afterwards** (Alerts → retention, in the UI —
it lives in the PVC, not Git) or the new store rebuilds toward the same cliff.

### What this was NOT

Worth recording, because the first diagnosis was wrong and cost a merge:

- **Not page-cache starvation.** The 1Gi → 2Gi bump in PR #118 did not fix it;
  it only bought ~4 more minutes per cycle. `metrics.db` at 361MB had plenty of
  room under 2Gi. The 450↔590Mi sawtooth on the day looked like cache eviction
  but was ordinary behaviour.
- **Not the SSH temperature collection** that floods the log at `logLevel: warn`.
  `temperatureMonitoringEnabled` has been true since 2026-07-25 and connection
  counts stay at 0 — the sessions are transient, not leaked.
- **Not the notification queue.** `notification_queue.db` is 23MB and static.
  (There *was* a real, separate problem there: dead-lettered Discord deliveries
  since 2026-08-29. Fixed 2026-09-23, see
  [below](#notification-dead-letters--the-discord-template).)
- **Not node memory.** See [server-01's stale kubelet capacity](#a-note-on-kubectl-top-nodes).
- **Not the agent.** `pulse-agent` buffers 60 reports and re-floods them at
  startup, which adds load to an already-failing server, but it does not cause
  the runaway. Scaling it to 0 still gives a quieter recovery.

### A note on `kubectl top nodes`

While chasing this, `kubectl top nodes` showed **k3s-server-01 at 100% memory**.
That is an artifact: kubelet caches machine info at startup and server-01 still
advertises `15210032Ki` (14.5GiB) while `/proc/meminfo` on the box reads 22.4GiB
with ~12GiB available, `MemoryPressure=False`. server-02 correctly reports
`23497324Ki`. It needs a k3s restart to clear, and strands ~8GiB from the
scheduler until then. There is no SSH from the workstation to the nodes — read
host state through the node-exporter pod:

```sh
kubectl exec -n monitoring <node-exporter-pod> -- head -5 /host/proc/meminfo
```

### Sizing, as it now stands

The limit is **2Gi** and the PVC **4Gi** (PR #118). Keep both: the PVC genuinely
was about a week from full at ~60Mi/day, and 2Gi leaves plenty of headroom over
the warm baseline. Just do not mistake either for the fix.

```sh
# should settle a few hundred Mi and stay flat, not climb steadily
kubectl top pod -n pulse -l app.kubernetes.io/name=pulse
# the two stores that matter
kubectl exec -n pulse deploy/pulse -- sh -c 'ls -l /data/metrics.db /data/alerts/events.db; du -sh /data'
```

**Known-good baselines**, measured from Prometheus. Use these, not impressions:

| | healthy (since 2026-09-23) | Aug 29 – Sep 21 | during the 09-21 loop | 09-23 CPU burn |
|---|---|---|---|---|
| CPU | **~0.25 cores** | ~1.1 cores, flat ±0.1 | 2.5 climbing, 6-9.5 pinned | ~3.1 cores, flat |
| memory | **~300Mi** warm | ~650Mi warm | constant climb to the limit | ~400Mi, flat |
| restarts | 0 | 0 | every 4-9min | rare OOM |

The 0.25-core figure was measured with the CPU evaluation window off, a fresh
`audit.db` and a 20MB `events.db`. Nobody knows what the window was set to
during the ~1.1-core weeks, so don't treat that number as the target.

Relapse = memory climbing steadily from a cold start, or CPU parked well above
~0.3 cores once warm. Check in this order:
1. The CPU evaluation window is still `0`.
2. `events.db` size.
3. `metrics.db` size.

For anything else, take a CPU profile ([how](#profiling-pulse)) before guessing.

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="pulse",container="pulse"}[10m]))
max(container_memory_working_set_bytes{namespace="pulse",container="pulse"})
```

Note that CPU stays elevated for some minutes after any restart — the startup
retention DELETE, an `incremental_vacuum`, and the agent replaying its 60
buffered reports all land at once. Judge it warm, not at t+2min.

---

## The 2026-09-23 CPU burn — rolling-CPU evaluation window

**Root cause: the v6.4 rolling-CPU alert window.** With
`metricEvaluationWindows: {all: {cpu: 300}}`, every agent report re-evaluates
every resource. Each evaluation calls
`ResourceRegistry.List`, which deep-clones and sorts **all** resources, so the
cost grows with the square of the resource count. Here that is ~755 resources,
519 of them Kubernetes objects (231 ReplicaSets alone). Fixed by setting the
window to `0` ("Current value") via the Alerts settings, with no restart. CPU
fell from **~3.1 to ~0.25 cores** within two minutes.

- **When it started:** 2026-09-21 20:34 UTC, the same UI alert save that tipped
  the [OOM loop](#the-2026-09-21-oom-loop--unbounded-alert-event-store). It then
  outlived that fix. Memory stayed fine (~400Mi of 2Gi), so the only symptoms were
  CPU flat at ~3.1 cores and a rare OOM. Whether that save switched the window
  on or only persisted the 300s default is unknown: Pulse does not audit
  alert-setting changes.
- **Profile (30s):** 68% of CPU in `ResourceRegistry.List`, reached via
  `UnifiedAgentHandlers.HandleReport → … → alerts.evaluateMetricWindow →
  Monitor.metricWindowPoints → MetricsTargetForResource`. After the change,
  `evaluateMetricWindow` is gone from the profile entirely.
- **Cost of the fix:** CPU alerts fire on the current value instead of a
  5-minute average, so expect a little more flapping on bursty VMs.
- **Upstream:** not reported as of 2026-09-23. #2146 looks similar (CPU pegged,
  slow metrics writes) but its profile is a different path (an `events.db`
  walk). No stable release after `v6.4.1` yet, only `v6.4.5` release candidates.
  **Re-test the window after any bump** by setting it back to 300 and
  profiling. Don't assume a new version fixed it.

**What this was NOT: the notification dead letters.** That was the obvious
suspect because the UI was showing a "Notification delivery needs attention"
warning. Clearing all 603 dead letters, the audit log and the incident `.bak`
files, then restarting, brought CPU straight back to ~3 cores. Those were real
problems, just not this one.

### Notification dead letters — the Discord template

The only webhook, "Discord - Saturn Notifier", is `service: generic` with a
custom template that pasted values in raw: `"description": "{{.Message}}"`.
TrueNAS replication alerts read `Replication "30mins replication task"
succeeded.`, and the embedded quotes produced invalid JSON (`invalid character
'3' after object key:value pair`) every 30 minutes. Alerts grouped into the
same message died with it. The result was 602 dead letters since 2026-08-29
(the v6.4.1 upgrade), at a flat ~72 failed attempts a day.

Fixed by adding `| jsonString` to `.Message`, `.ResourceName`, `.Node` and
`.Level`, the same way Pulse's built-in Discord template does
(`GET /api/notifications/webhook-templates` shows it). **Any future custom
template must escape every interpolated string the same way.**

Clearing the backlog uses the UI's own Dismiss action, which touches history
only: `POST /api/notifications/terminal-failures/dismiss`, no body. Dismissed
entries show up afterwards as `cancelled` in `/api/notifications/queue/stats`.

### The audit log is mostly noise

`/data/audit/audit.db` had reached **272MB, 99.98% of it `agent_config_fetch`
rows**: each agent fetching its config once a minute, ~7,200 rows a day. It
was deleted on 2026-09-23 (move the three files aside, delete the pod, then
remove them; Pulse recreates it empty). No retention setting for it has been
found, so expect it to regrow at ~4.5MB/day and repeat the cleanup if `/data`
gets tight.

### Profiling Pulse

`/debug/pprof/` is compiled in and admin-gated; a 401 means auth, not absence.
The pod's own `PULSE_AUTH_USER`/`PULSE_AUTH_PASS` work as Basic auth:

```sh
kubectl -n pulse exec deploy/pulse -- sh -c \
  'A=$(printf "%s:%s" "$PULSE_AUTH_USER" "$PULSE_AUTH_PASS" | base64 | tr -d "\n");
   wget -qO- -T60 --header "Authorization: Basic $A" \
     "http://localhost:7655/debug/pprof/profile?seconds=30"' > cpu.pprof
go tool pprof -top -cum cpu.pprof      # no binary needed; symbols are in the profile
```

The same Basic-auth header works for the JSON API (`/api/alerts/config`,
`/api/notifications/*`). BusyBox `wget` can't send PUT, so writes go through
`kubectl port-forward` and `curl -u`. The webhook GET returns the real URL,
unmasked, so a read-modify-write PUT is safe. Keep a copy of the object before
writing anyway.

## Troubleshooting

- **CPU flat at ~3 cores, memory fine, no restarts** — the rolling-CPU
  evaluation window is back on. Check
  `grep -o '"metricEvaluationWindows":[^]]*}}' /data/alerts.json` in the pod; it
  must read `{"all":{"cpu":0}}`. A missing key means the 300s default. See
  [the 2026-09-23 CPU burn](#the-2026-09-23-cpu-burn--rolling-cpu-evaluation-window).
- **"Notification delivery needs attention" / dead-lettered deliveries** —
  read the terminal error first: `GET /api/notifications/dlq` shows
  `lastError`. `template produced invalid JSON` means a custom webhook
  template is interpolating a value without `| jsonString`. Fix the template,
  *then* dismiss the backlog, or it simply refills.

- **Agent flaps online/offline on the dashboard** — more than one pod is
  reporting the same cluster under one `PULSE_AGENT_ID`. The Kubernetes agent is a
  **single-replica Deployment** for exactly this reason; don't scale it up or
  convert it back to a DaemonSet. (Original root cause of the post-deploy flap.)
- **k3s cluster flashes in/out of the dashboard every ~30s after any UI config
  save, and `/api/state` shows `"kubernetesClusters": []`** — upstream
  [#1558](https://github.com/rcourtman/Pulse/issues/1558): `Router.SetMonitor`
  never rebound `kubernetesAgentHandlers`, so after a config reload the reports
  kept landing in an orphaned monitor while the live one stayed empty. Restarting
  the pod cleared it until the next save. **Fixed upstream in v6.1.1**
  (maintainer, 2026-07-23). If it ever comes back: `restartCount` stays 0, agent
  logs show HTTP 200s, and `/api/state` is the tell — the flap is server-side,
  not the agent. **Retested on v6.1.1 2026-07-25** (throwaway server, emptyDir
  `/data`, real cluster reporting): a monitor reload now empties the cluster for
  **one agent report interval and no more** — 19s, then 57 consecutive 5s
  samples steady at 1 cluster / 3 nodes / 118 pods. Two notes for any future
  retest: v6's `/api/state` no longer has a `kubernetesClusters` key at all
  (unified `resources`, so count `.resources[] | select(.type=="k8s-cluster")`),
  and the reload is triggered by a **node** save — `POST /api/config/nodes`;
  `/api/config/system` answers 405 to POST/PUT/PATCH even with a valid admin
  session + CSRF, and the UI bundle never calls it for writes. The server log
  line that marks the reload is `monitoring loop stopped`.
- **Agent: `API token is already in use by agent "mac-…"`** — `PULSE_AGENT_ID` is
  missing/blank. With the single Deployment it's set to `safeqbit-local-hq`; if
  you ever fan out, every reporter must share that one ID.
- **Agent log: `Failed to fetch remote config … 403 Forbidden`** — harmless. The
  agent tries to pull an upstream remote config; it falls back to local defaults.
  Not a connection problem with the Pulse server.
- **Agent not showing in UI** — check `kubectl -n pulse logs deploy/pulse-agent`
  for a `PULSE_URL`/token error; confirm the pod can reach
  `pulse.pulse.svc.cluster.local:7655`.
- **Cert `pulse-tls` stuck `READY=False`** — DNS-01 challenge in progress
  (`delayBeforeChecks`); usually issues in 2–5 min. `kubectl -n pulse get
  challenge`.
- **Server OOMKilled in a loop, readiness timing out, CPU pegged at 3+ cores**
  — the alert event store has outgrown what the alert engine can process.
  Confirm with the two signatures: memory climbs at a constant rate from a cold
  start, and time-to-OOM scales linearly with the memory limit. Then check
  `ls -l /data/alerts/events.db`. See
  [the OOM loop](#the-2026-09-21-oom-loop--unbounded-alert-event-store).
  Raising the limit and bouncing the pod both only buy minutes.
- **Server CrashLoop after image bump** — a new tag may have migrated `/data`.
  Roll the image back in `04-deployment.yaml`; restore the PVC from Velero
  `pulse-weekly` if `/data` was corrupted.
- **Forgot/lost admin password** — re-seal a new `PULSE_AUTH_PASS` (kubeseal vs
  the controller in `kube-system`) and roll the Deployment, or change it in
  Settings → Security while logged in.

## Restore (DR)

Everything that defines the monitoring stack — config + encrypted target
credentials — is in the `pulse-data` PVC.

```sh
velero backup get | grep pulse-weekly
velero restore create --from-backup pulse-weekly-<TIMESTAMP>
```

Then reconcile Flux so the Deployment/Service/Ingress/DaemonSet match Git again.
The DaemonSet's token is sealed in Git; only the UI-entered Proxmox/Docker target
creds depend on the PVC restore.
