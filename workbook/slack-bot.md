# Cluster Slack Bot — on-demand queries via `/cluster`

**Created:** 2026-07-04. **v4:** 2026-07-07 (triage `why`, log `prev`/`grep`,
Velero per-backup detail, Longhorn/storage/jobs/silences/etcd/dns, grouped
help, formatting pass). **+ps** same day (task-manager view, PR #42).
**2026-07-08:** `ps` gained a NODE column (pod→node placement) and the bot
went to **2 replicas** for node-loss HA (see "Replicas & HA" below).

> **Usage playbooks** — which commands to run for which real incident,
> and where the bot hands off to a terminal — live in the companion
> **[slack-bot-manual.md](slack-bot-manual.md)**. This file covers
> architecture, Slack app setup, and ops.

A Slack bot for read-only cluster queries from Slack. Runs in
**Socket Mode**: each pod opens an outbound WebSocket to Slack and receives
slash commands over it — nothing inbound, the cluster stays LAN-only.
Free-tier Slack feature (no subscription; counts as 1 of the free plan's
10 workspace integrations, alongside the Alertmanager webhook).

**Replicas & HA (2026-07-08):** 2 replicas + RollingUpdate. Safe because
Socket Mode delivers each payload to exactly **one** of an app's open
connections (its documented HA mechanism, up to 10) — replicas do not
double-answer — and the bot has no self-initiated posting (the scheduled
reports are separate CronJobs). Motivated by the 2026-07-08 server-01
outage, which took the then-single bot down exactly when it was needed.
If double answers ever DO appear, that assumption broke: drop back to
`replicas: 1` and investigate before scaling again.

| | |
|---|---|
| Namespace | `monitoring` |
| Manifest | `infrastructure/.../configs/cluster-slack-bot.yaml` |
| Tokens | SealedSecret `cluster-slack-bot-tokens` (raw values in Vaultwarden) |
| Image | `python:3.13-alpine` + `pip install slack-bolt` (pinned) at start |
| Replicas | 1, `Recreate` — two bots would answer every command twice |

## Commands

```
/cluster summary                         daily health digest, on demand
/cluster backups                         weekly backup digest, on demand (~30s, B2 listing)
/cluster pods [ns]                       pod health; all-ns overview or one-ns full listing
/cluster deploys [ns]                    deployments/statefulsets/daemonsets ready-vs-desired
/cluster nodes [name]                    fleet view; with a name (prefix ok): conditions,
                                         taints, requests-vs-allocatable headroom meters
/cluster logs <ns> <pod> [ctr] [lines]   log tail (pod name PREFIX is enough; default 30, max 200)
             [prev] [grep <pattern>]     prev = crashed instance's logs; grep filters
                                         case-insensitively over the last 1000 lines
/cluster describe <ns> <pod>             pod detail: containers, restarts w/ last exit, its events
/cluster why <ns> <pod>                  ONE-SHOT TRIAGE: state + last exit + conditions +
                                         events + crashed container's previous log tail
/cluster events [ns]                     recent Warning events, newest first
/cluster ps [ns] [cpu|mem|net|io] [n]    TASK MANAGER: per-workload usage table (per-pod
                                         when a ns is given) — CPU, mem working set, net
                                         and disk I/O rates + NODE placement (common
                                         hostname prefix stripped: k3s-server-01 → 1;
                                         "all" = on every node) in a monospace code
                                         block; aliases taskmgr/htop/procs/util
/cluster top [ns]                        top-10 CPU / memory pods (Prometheus)
/cluster restarts [ns]                   pods restarted in the last 24h (Prometheus)
/cluster flux                            Kustomizations + HelmReleases ready/suspended + revision
/cluster velero [n | name]               last n backups (default 12); with a name (prefix ok):
                                         errors, failureReason, expiry, per-volume PVB detail
/cluster certs                           cert-manager expiries, soonest first (red <7d, amber <21d)
/cluster cnpg                            DB clusters: ready, primary, replication lag, last backup
/cluster pvcs [ns]                       volume claims grouped by ns: status, capacity, class
/cluster storage [n]                     n fullest volumes by used % (kubelet stats; red ≥90%)
/cluster longhorn                        volume robustness + per-node disk used/scheduled
                                         vs the over-provisioning limit
/cluster jobs [ns]                       cronjob last run/success + lingering Failed jobs
                                         (catches stale kopia-maintain jobs)
/cluster silences                        active Alertmanager silences, soonest expiry first
/cluster etcd                            Layer 0 check: etcd snapshots in B2 + node-local
                                         (red if newest B2 snapshot older than 8d)
/cluster dns <name>                      resolve a hostname from inside the cluster
/cluster ingresses                       hostname → service map
/cluster alerts                          currently firing Prometheus alerts
/cluster help                            all of the above, grouped, w/ kubectl equivalents
```

Singular/plural aliases work (`pod`, `deploy`, `log`, `event`, `pvc`,
`ingress`…), plus `triage`/`debug`→why, `volumes`/`lh`→longhorn,
`usage`/`disk`→storage, `cron`→jobs, `nslookup`/`resolve`→dns,
`snapshots`→etcd. `help` shows each command with the kubectl command
you'd type on a terminal.

v4 caveats worth remembering:

- `ps` groups pods into workloads via the kube-prometheus-stack recording
  rule `namespace_workload_pod:kube_pod_owner:relabel`; bare pods (e.g.
  Longhorn instance-managers) appear as their own rows. The NODE column
  reads the `node` label straight off the cAdvisor series the cpu/mem
  queries aggregate (no new RBAC). Don't join via `kube_pod_info` for
  this: KSM drops deleted pods instantly while cAdvisor series linger
  ~5 min in instant queries, so pod churn shows "?" NODEs (the v1 bug). hostNetwork pods
  (metallb-speaker, node-exporter, flannel-offload…) report **whole-node**
  NIC traffic as their NET/s. DISK/s is cAdvisor container-fs I/O — writes
  to PVC block devices aren't always attributed. It renders in a code
  block, so no emoji/colour inside the table (Slack limitation).

- `dns` resolves in the **bot pod, which runs ndots=1** — it will NOT
  reproduce the ndots:5 search-domain flapping that default app pods hit
  (see the DNS-amplification notes). It answers "does this name exist in
  cluster DNS", not "will every pod resolve it".
- `velero <name>` per-volume detail comes from PodVolumeBackup CRs, which
  are pruned with their backup — older backups show CR-level status only.
  Full logs still need `velero backup logs` in the velero pod.
- `alerts` reads Prometheus `ALERTS`, which does not know about
  Alertmanager silences — a silenced alert still shows as firing there;
  cross-check with `silences`.
- `silences` is read-only (GET on the Alertmanager API). Creating or
  expiring silences stays in the Alertmanager UI, on purpose.

Replies go through the slash command's `response_url` → visible **only to
the invoker**, in whatever channel the command was typed. `summary` and
`backups` execute the existing report ConfigMaps
(`cluster-summary-report-script`, `backup-summary-report-script`) as
subprocesses with `SLACK_WEBHOOK_URL` overridden to the `response_url` —
the CronJobs and the bot share one copy of the report code.

## Security posture

- **Read-only by design.** The bot's RBAC is `get,list` on pods, pods/log,
  nodes, events, PVCs, deployments/statefulsets/daemonsets, ingresses,
  Flux kustomizations/helmreleases, cert-manager certificates, CNPG
  clusters, batch jobs/cronjobs, Longhorn volumes/nodes/settings, k3s
  etcdsnapshotfiles and velero podvolumebackups, plus reuse of the
  `backup-summary-report` ClusterRole and the velero `cloud-credentials`
  read Role. No write verbs anywhere — a leaked Slack token can look at
  the cluster, never touch it. Do not add restart/delete commands.
- `dns` was deliberately NOT extended to a `curl <url>` command — that
  would turn a leaked Slack token into a LAN request proxy (SSRF).
- `logs` ships pod log tails into Slack (ephemeral + invoker-only, but
  Slack-hosted) — mind apps that log secrets.
- `allowed-user-ids` key in the SealedSecret (comma-separated Slack user
  IDs) restricts who may invoke; empty/absent = anyone in the workspace.

## Slack app config (UI-only — recreate at api.slack.com/apps)

1. **Create App → From scratch**, name `safeqbit-cluster-bot`, pick the
   workspace.
2. **Socket Mode** → enable → generate an **app-level token** with scope
   `connections:write` → this is the `xapp-…` token.
3. **Slash Commands → Create New Command**: command `/cluster`, short
   description "read-only k3s cluster queries", usage hint
   `summary | backups | pods [ns] | nodes | alerts`. (Socket Mode ⇒ the
   Request URL field is not used.)
4. **OAuth & Permissions** → Bot Token Scopes: `chat:write` (plus
   `commands`, added automatically by step 3) → **Install to Workspace** →
   this is the `xoxb-…` bot token.
5. Reseal `cluster-slack-bot-tokens` with keys `bot-token`, `app-token`,
   `allowed-user-ids` (see maintenance.md → Sealed Secrets), store raw
   values in Vaultwarden.

## Ops

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=cluster-slack-bot
kubectl -n monitoring logs deploy/cluster-slack-bot --tail=50
# healthy startup ends with a Bolt "connection established" / apps run log
# Restart (reconnect, or pick up a bot.py ConfigMap change — pods don't
# auto-restart on ConfigMap edits): delete pods one at a time. Do NOT use
# `rollout restart` — it edits the pod template (restartedAt annotation),
# which Flux's server-side apply strips on the next reconcile, rolling
# everything a second time (observed 2026-07-08).
kubectl -n monitoring delete pod <bot-pod-1>   # wait for its replacement to go 1/1…
kubectl -n monitoring delete pod <bot-pod-2>   # …then the other, so one bot stays connected
```

- The pod pip-installs `slack-bolt` at start — a PyPI outage delays
  (re)starts but never affects a running bot.
- Slack's `response_url` allows 5 posts within 30 min per invocation —
  each command uses 1–2, so this never binds in practice.
- Token rotation: regenerate in the Slack app UI, reseal, restart the
  deployment.
- **After ANY change to the ConfigMap**: the pod only reads `/bot/bot.py`
  at start — Flux applying the manifest is not enough, restart the
  deployment.

### Testing handlers without Slack (the harness pattern)

Every command handler is a plain function `def cmd(response_url, args)`
whose only side effect is `post()`. That makes the whole bot testable
from a workstation with cluster read access, no Slack round-trip:

```python
# extract bot.py from the ConfigMap, then:
src = open("bot.py").read()
g = {}
exec(src.split("app = App(")[0], g)      # defs only, no Slack bootstrap
                                          # (stub slack_bolt in sys.modules,
                                          #  set KUBERNETES_SERVICE_HOST)
g["k8s_raw"] = lambda p, timeout=30: subprocess.run(
    ["kubectl", "get", "--raw", p], capture_output=True, text=True,
    check=True).stdout                    # my creds, read-only verbs only
g["k8s"] = lambda p: json.loads(g["k8s_raw"](p))
g["promq"] = ...   # kubectl get --raw the Prometheus service /proxy path
g["post"] = lambda rurl, text: print(text)
g["why"]("fake-url", ["monitoring", "grafana"])
```

Prometheus/Alertmanager go through the apiserver service proxy
(`/api/v1/namespaces/monitoring/services/<svc>:<port>/proxy/…`). This is
how v4 and `ps` were verified against the live cluster before merging —
it caught three real bugs (duplicate wrong-namespace kubelet volume
series, k3s vendor node-conditions misread, a blank-line glitch).
One gap to know about: the shim raises `CalledProcessError` where the
real `k8s_raw` raises `urllib.error.HTTPError`, so error-path handling
(e.g. `logs prev` on a pod with no previous instance) needs a mental
translation when it "fails" in the harness.
