# Cluster Bot User Manual — `/cluster` vs the real commands

**Created:** 2026-07-07. Companion to [slack-bot.md](slack-bot.md) (which
covers architecture, Slack app setup and ops). This document is about
*using* the bot: what to type when something breaks, what the output
means, and where the bot stops and the terminal begins.

**The mental model:** the bot is the **first responder** — it answers
"what is broken and why" from a phone, without VPN or a laptop, in under
a minute. It is read-only by design, so **every fix still ends at a
terminal**. The playbooks below map each real incident this cluster has
had to (a) the bot commands that diagnose it and (b) the real commands
that fix it.

---

## The first 60 seconds (any alert, any weirdness)

```
/cluster alerts          what is Prometheus complaining about?
/cluster summary         overall state — same digest the morning CronJob sends
/cluster pods            anything not Running/Completed, cluster-wide
/cluster events          recent Warning events, newest first
```

Then narrow down:

```
/cluster why <ns> <pod>       ONE command: state, last exit code, unmet
                              conditions, events, crash logs. Start here.
/cluster logs <ns> <pod> prev crashed instance's logs (the current
                              instance hasn't failed yet — the reason is
                              always in the PREVIOUS one)
/cluster logs <ns> <pod> grep <pattern>
/cluster ps [ns]              task manager: who is eating cpu/mem/net/disk
```

Pod names accept a unique prefix everywhere — `/cluster why monitoring
grafana` is enough, nobody types hash suffixes on a phone.

---

## Playbooks from real incidents

### 1. Pod crash-looping (the Grafana SQLite incident, pre-2026-05)

The pattern that killed old Grafana: crash loop, and the *current* logs
look innocent because the process dies before logging the cause.

| Step | Bot | Terminal |
|---|---|---|
| See it | `/cluster restarts` → `/cluster why <ns> <pod>` | `kubectl describe pod`, `kubectl get events` |
| Root cause | `why` shows last exit code + previous-instance log tail automatically; deeper: `/cluster logs <ns> <pod> prev 100` | `kubectl logs -p`, exit codes in `describe` |
| Fix | — bot stops here | restart: `kubectl -n <ns> rollout restart deploy/<name>`; if Flux fights you, suspend first — see maintenance.md → Flux suspend/resume |

Verdict shortcuts `why` gives you: `CrashLoopBackOff` + exit code 1 →
app config/DB problem (read the prev logs); `OOMKilled` (exit 137) →
check `/cluster ps <ns>` mem vs limits; `ImagePullBackOff` → typo'd tag
or registry down; `CreateContainerConfigError` → missing
Secret/ConfigMap (check `/cluster events <ns>`).

### 2. Grafana "no data" / volume full (Prometheus TSDB, 2026-06-28)

Prometheus filled its PVC to 100% and Grafana went blank. The bot now
sees this *before* it bites:

| Step | Bot | Terminal |
|---|---|---|
| Detect | `/cluster storage` — fullest volumes with ▰▰▰ meters, red ≥90% | `kubectl -n monitoring exec prometheus-… -- df -h /prometheus` |
| Confirm consumer | `/cluster ps monitoring` (DISK/s column), `/cluster why monitoring prometheus` | `du` inside the pod |
| Fix | — | retention caps live in `controllers/monitoring.yaml` (retentionSize, added after this incident); PVC expansion via Longhorn — but read project memory first: expansion wedged once on the stale-iscsid bug, fixed by deleting the volume's **Engine CR** (replicas/data safe) |

Weekly habit even without alerts: `/cluster storage` + `/cluster
longhorn`. The over-provisioning headroom on the longhorn view is the
early warning the 06-28 incident didn't have.

### 3. Velero backup PartiallyFailed (2026-07-06 runbook)

Full runbook: [velero-partial-failed-cleanup.md](velero-partial-failed-cleanup.md).
The bot covers the *triage* half:

| Step | Bot | Terminal |
|---|---|---|
| Which backup, how bad | `/cluster velero` → `/cluster velero <name>` (prefix ok): errors, failureReason, per-volume PodVolumeBackup failures | `kubectl -n velero get backups.velero.io`, `kubectl -n velero describe backup <name>` |
| Full log | — (bot shows PVB messages only) | `kubectl -n velero exec deploy/velero -- /velero backup logs <name>` |
| Stale kopia-maintain jobs | `/cluster jobs velero` — lingering Failed jobs flagged "stale?" | `kubectl -n velero delete job <name>` |
| Clear the 16d alert | — (needs a write) | new success from the same schedule: `kubectl -n velero exec deploy/velero -- /velero backup create --from-schedule=<sched>`; **remember: restarting velero does NOT clear the alert** |
| Orphaned schedule | `/cluster silences` — check the silence exists and hasn't lapsed | create silence in Alertmanager UI until failure+16d |
| Delete a bad backup | — | `exec … /velero backup delete <name> --confirm` — **never `kubectl delete`**, backup-sync resurrects the CR |

### 4. Longhorn volume degraded / node disk trouble

(multipathd blacklist 2026-05-18, stale-iscsid engine wedge 2026-06-28,
near-over-provisioning scare)

| Step | Bot | Terminal |
|---|---|---|
| Detect | `/cluster longhorn` — degraded/faulted volumes named with their PVC, per-node used% + scheduled% vs limit | Longhorn UI, `kubectl -n longhorn-system get volumes.longhorn.io` |
| Which app is on it | volume rows show `ns/pvcName` directly | `kubectl get pvc -A \| grep <pvc-…>` |
| Fix | — | rebuilds usually self-heal; a volume wedged after node reboot → check for the stale-iscsid-PID pattern, fix was deleting the volume's Engine CR (memory: `project_prometheus_full_longhorn_expansion`); mounts failing on a fresh/rebuilt node → multipath blacklist missing, see [node-bootstrap.md](node-bootstrap.md) |

### 5. "Pods can't talk across nodes" (flannel VXLAN offload, 2026-05-27)

The signature: after a Proxmox node reboot, cross-node TCP/UDP dies but
**ICMP works**, apps time out weirdly, DNS is flaky.

| Step | Bot | Terminal |
|---|---|---|
| Suspect it | `/cluster events` full of probe timeouts on ONE node's pods; `/cluster nodes <name>` fine otherwise; `/cluster dns <name>` works (bot pod may be on a healthy node) | `curl` between pods on different nodes |
| Confirm + fix | — | on the rebooted node: `systemctl status flannel-fix-offload.service`; manual: `ethtool -K flannel.1 tx-checksum-ip-generic off`. Unit should self-run at boot — see [node-bootstrap.md](node-bootstrap.md) |

Bot tell-tale: symptoms concentrated on pods of one node right after
that node rebooted → think offload before anything else.

### 6. Name resolution flapping (ndots:5 amplification)

| Step | Bot | Terminal |
|---|---|---|
| Check the name exists | `/cluster dns <fqdn>` | `kubectl run -it --rm dbg --image=busybox -- nslookup <name>` |
| Interpret | **caveat:** the bot pod runs ndots=1, so a name resolving in the bot does NOT prove app pods resolve it reliably — 4-dot FQDNs still flap under default ndots:5 | reproduce inside the affected pod |
| Fix | — | short service names, or trailing-dot absolute FQDNs (memory: `project-dns-search-amplification`) |

### 7. Flux drift / stuck reconcile / suspended-and-forgotten

| Step | Bot | Terminal |
|---|---|---|
| Status | `/cluster flux` — Ready/Suspended per Kustomization + HelmRelease, revision, failure message inline | `kubectl get kustomizations,helmreleases -A` |
| Force sync | — | annotate `reconcile.fluxcd.io/requestedAt="$(date +%s)"` on the gitrepository, then the kustomization (flux CLI not installed on the workstation) |
| Suspend/resume | — | maintenance.md → Flux suspend/resume section |

The FluxSuspended info-alert nags daily; `/cluster flux` shows the ⏸
rows it means.

### 8. Certificate expiry

| Step | Bot | Terminal |
|---|---|---|
| Check | `/cluster certs` — soonest first, red <7d, amber <21d, NOT READY flagged | `kubectl get certificates -A`, `kubectl describe certificaterequest` |
| Fix | — | usually cert-manager renews itself; failures are DNS-01/Cloudflare issues → `kubectl -n cert-manager logs deploy/cert-manager` |

### 9. Database (CNPG) health

| Step | Bot | Terminal |
|---|---|---|
| Fleet view | `/cluster cnpg` — ready/instances, current primary, replication lag, last backup age | `kubectl get clusters.postgresql.cnpg.io -A` |
| Suspicious lag / failover | `/cluster why <ns> <cluster>-N` on the lagging instance | maintenance.md → CNPG connect/dump for psql access and dumps |
| Backup missing | `/cluster cnpg` (last-backup age) + `/cluster backups` weekly digest | see [cnpg-strategy.md](cnpg-strategy.md) + [backup-strategy.md](backup-strategy.md) |

### 10. "Are the backups actually running?" (all 4 layers)

Layer map lives in [backup-strategy.md](backup-strategy.md). Bot
coverage per layer:

```
Layer 0  etcd → B2        /cluster etcd      red if newest B2 snapshot >8d
Layer 1  Longhorn         /cluster longhorn  volume health (schedule: UI/Git)
Layer 2  CNPG snapshots   /cluster cnpg      last-backup age per DB
Layer 3  Velero → B2      /cluster velero    phases + errors, drill into one
   all                    /cluster backups   the full weekly digest, on demand
```

Restores are terminal-only, always: [restore-drills.md](restore-drills.md).

### 11. Node acting up

| Step | Bot | Terminal |
|---|---|---|
| Fleet | `/cluster nodes` — readiness + live CPU | `kubectl get nodes -o wide` |
| One node | `/cluster nodes <name>` — pressure conditions, taints, requests-vs-allocatable headroom | `kubectl describe node <name>` |
| Who's heavy on it | `/cluster ps` (workload view) or `/cluster top` | `kubectl top pods -A --sort-by=cpu` |
| Fix | — | drain/reboot per maintenance.md; after ANY node reboot check playbooks 4 & 5 above (multipath, flannel offload) — both are reboot-triggered |

### 12. Resource hogs / capacity questions

| Question | Bot |
|---|---|
| What's eating CPU right now? | `/cluster ps` (default sort) |
| Memory? network? disk io? | `/cluster ps mem` / `net` / `io` |
| Inside one app? | `/cluster ps <ns>` — per-pod |
| Is a node schedulable-full? | `/cluster nodes <name>` headroom meters |
| Which volume fills next? | `/cluster storage` |
| Who restarted overnight? | `/cluster restarts` |

`ps` caveats: hostNetwork pods (metallb-speaker, node-exporter,
flannel-offload) show whole-node NIC traffic as NET/s; DISK/s is
container-fs I/O and doesn't always attribute PVC block writes.

---

## Where the bot deliberately stops

- **No writes, ever.** No restart, delete, scale, silence-create, exec.
  A leaked Slack token can look, never touch. This is a standing
  decision — don't "improve" it.
- **No `curl <url>` command** — that would be an SSRF proxy into the
  LAN. `dns` is the only network probe, and it only resolves names.
- **No secrets access** in RBAC, but `logs`/`why` DO ship log tails to
  Slack — apps that log tokens/passwords leak them to Slack's servers.
  (Ephemeral + invoker-only, but still Slack-hosted.)
- Output truncates: lists cap their rows, messages clip near Slack's
  ~40k limit. "… and N more" means go to a terminal.
- `alerts` reads Prometheus and does **not** know about Alertmanager
  silences — a silenced alert still shows as firing; cross-check
  `/cluster silences`.

## Quick reference: bot ↔ terminal

| Bot | Terminal equivalent |
|---|---|
| `/cluster pods [ns]` | `kubectl get pods -A` / `-n ns` |
| `/cluster deploys [ns]` | `kubectl get deploy,sts,ds -A` |
| `/cluster nodes [name]` | `kubectl get nodes -o wide` / `describe node` |
| `/cluster logs ns pod [prev] [grep p]` | `kubectl logs -n ns pod [-p] [\| grep p]` |
| `/cluster describe ns pod` | `kubectl describe pod -n ns pod` |
| `/cluster why ns pod` | describe + events + `logs -p` + restarts, combined |
| `/cluster events [ns]` | `kubectl get events -A --field-selector type!=Normal` |
| `/cluster top [ns]` / `ps` | `kubectl top pods -A` / htop-ish, no equivalent |
| `/cluster restarts [ns]` | `kubectl get pods -A \| grep -v " 0 "` (roughly) |
| `/cluster flux` | `kubectl get kustomizations,helmreleases -A` |
| `/cluster velero [n\|name]` | `kubectl -n velero get/describe backups.velero.io` |
| `/cluster certs` | `kubectl get certificates -A` |
| `/cluster cnpg` | `kubectl get clusters.postgresql.cnpg.io -A` |
| `/cluster pvcs [ns]` / `storage` | `kubectl get pvc -A` / kubelet volume stats |
| `/cluster longhorn` | `kubectl -n longhorn-system get volumes.longhorn.io` + Longhorn UI |
| `/cluster jobs [ns]` | `kubectl get cronjobs,jobs -A` |
| `/cluster silences` | Alertmanager UI → Silences |
| `/cluster etcd` | `kubectl get etcdsnapshotfiles.k3s.cattle.io` |
| `/cluster dns name` | in-pod `nslookup` (but see ndots caveat) |
| `/cluster ingresses` | `kubectl get ingress -A` |
| `/cluster alerts` | Prometheus UI → Alerts |
