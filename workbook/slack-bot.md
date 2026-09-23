# Cluster Slack Bot — on-demand queries via `/cluster`

**Created:** 2026-07-04. **v4:** 2026-07-07 (triage `why`, log `prev`/`grep`,
Velero per-backup detail, Longhorn/storage/jobs/silences/etcd/dns, grouped
help, formatting pass). **+ps** same day (task-manager view, PR #42).
**2026-07-08:** `ps` gained a NODE column (pod→node placement) and the bot
went to **2 replicas** for node-loss HA (see "Replicas & HA" below).
**2026-07-19:** `wiki` / `man` — a static knowledge base (kubectl man
pages, troubleshooting methodologies, cluster-specific KB) served from
its own ConfigMap (see "The wiki" below).
**2026-07-21:** bot.py gained a built-in **local mode** (`python3 bot.py
<cmd>` off-cluster runs any handler with kubectl credentials — the old
exec-and-monkeypatch harness, productized) and a companion **TUI**
(see "Local mode & the TUI" below).
**2026-07-22:** the TUI became **QubeCommander**
(`~/git/qube-commander/qube`, renamed from `~/git/cluster-bot/tui.py`
and now git-versioned): an htop-style terminal commander whose trace
pane defaults to a **how-to view** — every call each command makes is
shown as the kubectl/grep (or Grafana → Explore) step a human would type
to get the same answer, F7 flips to the raw `kubectl get --raw` calls.

**2026-09-19:** `updates` — weekly version-drift digest
(`chart-updates-report.yaml`), on demand from the bot or Mondays 08:30
from its own CronJob. Landed as Helm charts, extended the same day to
Git-pinned container images and to digest drift on floating tags
(see "Updates" below).

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
/cluster ps [ns] [cpu|mem|net|io] [n|all] TASK MANAGER: per-workload usage table (per-pod
                                         when a ns is given) — NAMESPACE + WORKLOAD as
                                         separate columns (POD + WORKLOAD in ns mode),
                                         full names (never truncated), CPU, mem working
                                         set, net and disk I/O rates + NODE placement
                                         (common hostname prefix stripped:
                                         k3s-server-01 → 1; "all" = on every node) in a
                                         monospace code block. Shows top-n by sort key
                                         (default 20) with a visible "+N more" trailer;
                                         "all" lists everything — idle workloads sort
                                         last on cpu, they are NOT missing, just below
                                         the cut. Aliases taskmgr/htop/procs/util
/cluster top [ns]                        top-10 CPU / memory pods (Prometheus)
/cluster restarts [ns]                   pods restarted in the last 24h (Prometheus)
/cluster flux                            Kustomizations + HelmReleases ready/suspended + revision
/cluster updates                         version drift, three parts: HelmRelease chart versions,
                                         Git-pinned image tags (both vs newest stable upstream,
                                         sorted major→minor→patch with changelog links), and
                                         digest drift on floating tags
                                         (~7s; aliases upgrades/versions/charts)
/cluster velero [n | name]               last n backups (default 12); with a name (prefix ok):
                                         errors, failureReason, expiry, per-volume PVB detail
/cluster certs                           cert-manager expiries, soonest first (red <7d, amber <21d)
/cluster cnpg                            DB clusters: ready, primary, replication lag, last backup
/cluster pvcs [ns]                       volume claims grouped by ns: status, capacity, class
/cluster storage [n]                     n fullest volumes by used % (kubelet stats; red ≥90%)
/cluster longhorn                        volume robustness + per-node disk used/scheduled
                                         vs the over-provisioning limit
/cluster longhorn <pvc>                  one volume (PVC/volume-name prefix ok): state, node,
                                         spec-vs-on-disk size, replica placement, snapshot list
                                         (size/age/user-vs-system/removed)
/cluster longhorn snapshots              snapshot debt, ALL volumes sorted by bytes: count,
                                         bytes, "trapped" (markRemoved awaiting fold); ends
                                         with a plain-language trapped/fold legend
/cluster longhorn jobs                   RecurringJobs (task/cron/retain/groups + last snap age)
                                         + snapshot-related settings; aliases: retention, recurring
/cluster jobs [ns]                       cronjob last run/success + lingering Failed jobs
                                         (catches stale kopia-maintain jobs)
/cluster silences                        active Alertmanager silences, soonest expiry first
/cluster etcd                            Layer 0 check: etcd snapshots in B2 + node-local
                                         (red if newest B2 snapshot older than 8d)
/cluster dns <name>                      resolve a hostname from inside the cluster
/cluster ingresses                       hostname → service map
/cluster alerts                          currently firing Prometheus alerts
/cluster wiki [section|topic]            knowledge base: index, section listing, or one page
                                         (topic prefix ok; sections kubectl/method/cluster)
/cluster wiki search <term>              full-text search across all wiki pages
/cluster man <cmd>                       kubectl man page shortcut (man logs = wiki kubectl.logs)
/cluster help                            all of the above, grouped, w/ kubectl equivalents
```

Singular/plural aliases work (`pod`, `deploy`, `log`, `event`, `pvc`,
`ingress`…), plus `triage`/`debug`→why, `volumes`/`lh`→longhorn,
`usage`/`disk`→storage, `cron`→jobs, `nslookup`/`resolve`→dns,
`snapshots`→etcd, `upgrades`/`versions`/`charts`/`drift`→updates,
`kb`/`docs`/`howto`/`guide`/`runbooks`→wiki,
`manual`→man. `help` shows each command with the kubectl command
you'd type on a terminal.

## The wiki (`wiki` / `man`)

A static, in-Slack knowledge base — three sections, one ConfigMap key
per page, named `<section>.<topic>`:

- **`kubectl.*`** (14 pages) — man pages for the kubectl commands
  actually used here, each ending with its `/cluster` equivalent and
  any cluster-specific traps (`rollout restart` SSA double-roll,
  `backups.velero.io` name collision, GitOps edit warnings…).
- **`method.*`** (10 pages) — troubleshooting methodologies: incident
  triage order, pod crash/pending, DNS/network, storage, node-down,
  Flux sync, certs/ingress, databases, backup/restore.
- **`cluster.*`** (11 pages) — this cluster's facts: ASCII map, nodes,
  network, storage, backups, databases, monitoring, GitOps flow, app
  inventory, the gotchas hall of fame, and a runbook index.

Mechanics, all in `cluster-slack-bot-wiki.yaml`:

- Pages are **Slack mrkdwn** (no markdown headers/links/tables). The
  first line of each page is `*Title* — summary`; listings and search
  results show the summary part.
- Resolution: exact key → exact topic → prefix → substring; ambiguous
  queries list the candidates (e.g. `storage` matches both
  `method.storage` and `cluster.storage` — intentional).
- `man <x>` prefers `kubectl.<x>` and falls through to `wiki <x>`.
- The bot reads `/wiki` **per request**, and kubelet syncs ConfigMap
  volumes within ~1 min — **wiki edits go live without a pod restart**
  (bot.py itself still needs the one-at-a-time pod delete).
- The volume is mounted `optional: true`, so a missing wiki ConfigMap
  never blocks bot startup — `wiki` just answers "no pages found".
- Content is deliberately duplicated from README/workbook (Slack can't
  follow repo links). When a cluster fact changes, grep
  `cluster-slack-bot-wiki.yaml` too — the runbooks stay the source of
  truth, the wiki page points at them.

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
the invoker**, in whatever channel the command was typed. `summary`,
`backups` and `updates` execute the existing report ConfigMaps
(`cluster-summary-report-script`, `backup-summary-report-script`,
`chart-updates-report-script`) as subprocesses with `SLACK_WEBHOOK_URL`
overridden to the `response_url` — the CronJobs and the bot share one copy
of the report code.

## Updates (`updates`)

Answers one question: **what are we behind on, and how far.** Three parts —
Helm charts, Git-pinned container images, and floating-tag digest drift. Manifest
`configs/chart-updates-report.yaml` — ConfigMap script + its own
ServiceAccount/ClusterRole + a CronJob (Mondays 08:30 ET, 30 min after the
backup digest). The bot execs the same script for `/cluster updates`.
A full run is ~7s and ~5 KB of Slack message.

The file and its resources are still named `chart-updates-*` from when it
only did charts. That is deliberate: every layer below `flux-system` is
`prune: false`, so renaming them would leave the OLD CronJob and ConfigMap
running and you would get **two digests every Monday** until they were
deleted by hand — not a trade worth making for a filename.

**Version data comes from the upstream chart repos** — 10 `index.yaml`
files, ~7.6 MB, once a week over HTTPS. No credentials, and the public chart
repos do not rate-limit this. A repo that fails degrades to one "not
checked" line and costs the other nine nothing.

**Why not source-controller's cache** (it was the original design, and the
answer is worth keeping): Flux downloads all ten of those files anyway and
serves each at `.status.artifact.url`
(`http://source-controller.flux-system.svc.cluster.local./helmrepository/<ns>/<name>/index-<hash>.yaml`),
so reading that would mean zero egress. It is **unreachable from
`monitoring`, permanently.** Flux ships its own NetworkPolicies into
`flux-system`: `allow-egress` admits ingress only from pods already inside
`flux-system`, and `allow-scraping` opens port 8080 alone. The artifact
server listens on **9090**, so the request gets a TCP reset — `Connection
refused` on all ten repos, k3s's policy controller rejecting rather than
dropping. Opening it would mean a NetworkPolicy allowing
`monitoring → app=source-controller:9090`; **deliberately not done
(2026-09-19)** — poking a hole in Flux's own firewall is not worth saving
ten weekly requests for public files. The script still tries the cache
first, because a refused connection costs nothing and the report would
start using it with no code change if that policy were ever added. The
refusals are expected; don't "fix" them.

- **The index parser is hand-rolled**, because the report images are
  stdlib-only (no PyYAML, same rule as the sibling digests). A chart index
  is machine-generated and rigid — `entries:` at column 0, chart names at
  indent 2, one `- ` item per version with its fields at indent 4 — so the
  parser matches `version:`/`created:`/`home:` at **exactly indent 4**.
  That precision is what keeps an entry's nested `dependencies:` block
  (which has its own `version:` at indent 6) and `annotations:` block
  scalars out of the results. It was validated line-for-line against
  PyYAML on all ten chart repos in use (1138-version
  `kube-prometheus-stack` included) and streams, so a 6 MB index never
  lands in memory.
- **Severity is the semver bump, not a CVE score.** major → :red_circle:,
  minor → :warning:, patch → :arrow_up:, sorted in that order, then by how
  many stable releases you are behind. That measures how risky the *upgrade*
  is, which is not the same as how urgent it is — a patch can be the security
  fix and a major can be pure refactoring. Nothing in a chart index or a
  registry knows about vulnerabilities. Real CVE severity needs a scanner;
  evaluated and deferred, with the numbers and the trigger conditions in
  **[vulnerability-scanning.md](vulnerability-scanning.md)**.
- **Changelog links** come from the index entry's own `home:` field, so
  they cost nothing and cannot drift out of date.
- **Pre-releases never count as available** — `rc`/`alpha`/`beta`/`dev`/
  `pre`/`snapshot`/`nightly`/`next` are filtered out, so the report never
  suggests an rc.
- **`HOLDS` in the script is the pin list.** A chart listed there reports
  under *Held* with the reason instead of nagging. Today: `cert-manager`,
  held at v1.16.1 (July 2026 wave — 1.21.x has known crash-loop bugs,
  target 1.20.x if ever), and `longhorn`, held at 1.11.3 (1.12's internal
  NetworkPolicies block Prometheus — `longhorn-1.12-upgrade.md`). **Add to
  `HOLDS` whenever an upgrade is deliberately declined**, or the digest will
  argue every week with a decision already made.
- **Weekly, not daily**, on purpose: upgrades land in batches weeks apart,
  so a daily copy of the same six lines becomes wallpaper — and a noisy
  channel already cost this cluster once (the orphaned-kopia storm hit
  Slack's `message_limit_exceeded` and broke both report CronJobs). Type
  `/cluster updates` when you want it sooner.

### The image half

`IMAGES` in the script is an explicit map (image repo → source, repo, tag
pattern) and `IGNORE` an explicit prefix list of what not to track, each
with its reason. Anything **running** that matches neither is reported
under *Untracked images* — so a newly deployed app gets noticed instead of
quietly going unwatched. That is the whole reason the map is a list rather
than a guess. 17 images are tracked today.

- **The running version comes from the workloads, not from Git.** A tag
  bumped in a manifest but not yet reconciled should still read as pending.
- **Sources are per image, because the tag you paste into a manifest is not
  always the tag upstream names its release.** `github` (the releases API,
  which is also the changelog) is preferred *where the release tag and the
  image tag are the same string* — verified per image. `hub` (Docker Hub's
  tag list) covers the ones where they differ: pwpush releases `v2.13.0`
  but ships `2.13.0`; photoprism releases `260728-bbde8f452` but ships
  `260728`; netbox compounds two versions into `v4.7.1-5.1.1`; pangolin
  ships an `ee-postgresql-` variant. Those link to the releases *page*
  rather than a tagged release.
- **The `pat` regex matches the running tag AND the candidate tags.** That
  does two jobs: it filters pre-releases with no keyword blocklist (betas,
  rcs, `latest` and external-snapshotter's `client/v8.6.0` simply do not
  match), and it guarantees the version the report prints is a **real tag
  you can paste straight into the manifest**.
- **Flux is tracked as flux2**, the thing you actually upgrade, rather than
  as four controller images — reading the running version from
  source-controller's `app.kubernetes.io/version` label.
- **GitHub anonymous is 60 requests/hour** and a run makes ~13, so no token
  is needed. The optional `github-token` key in the bot's SealedSecret
  switches it on if `/cluster updates` is ever run by hand often enough to
  matter.

**Not version-tracked, on purpose.** The ~40 sidecar images the charts
themselves manage (Longhorn's CSI sidecars, cert-manager's three, the
prometheus-stack's six): you cannot move one without bumping its chart,
which the chart half already tells you. They sit in `IGNORE` with their
reason.

### The floating half

For a tag that carries no version — `affine:stable`, `redis:7-alpine`,
`postgres:16-alpine`, the alpine/python/node bases — "newer" means the tag
was rebuilt and now points at a different digest. Semver has nothing to
say. What *can* be said is whether the digest these pods are running is
still the one the tag resolves to: if not, **a restart would silently
change what runs**.

- **The digest comes from pod status**, the only place the digest actually
  pulled is recorded — a workload spec carries just the tag, which is the
  whole problem. Reading all pods is ~1.5 MB of JSON for 167 pods and ~33 MB
  peak RSS, comfortably inside the job's 128Mi limit.
- **Both sides are the multi-arch *index* digest**, so the comparison is
  apples-to-apples. That was worth verifying rather than assuming: if the
  kubelet reported a platform-specific manifest digest, every image would
  read as drifted forever. `alpine:3.20` is the control — it matched its
  registry digest exactly, while the others differed because the tags really
  had moved.
- **Only drift older than `FLOAT_STALE_DAYS` (30) is listed.** python, redis
  and postgres are rebuilt weekly, so a row per moved tag would be permanent
  wallpaper — the exact failure this report is built to avoid. Recent moves
  collapse into a single "routine base-image rebuilds" line. In practice
  that is 9 moved, 2 listed.
- **An unknown move date always lists.** Docker Hub reports when a tag last
  moved; a bare OCI `HEAD` does not. The only such image here is
  `affine:stable`, an *app* rather than a base image, where a move really
  does mean a new release worth acting on.
- **The set checked is derived from `IGNORE`** — an entry whose reason starts
  with `floating` is digest-checked — so there is one list to keep, not two.
  `ghcr.io/immich-app/postgres` is excluded by having a different reason: a
  bump there is a data migration, not a restart.

To pick a moved tag up: **delete the pod**. Not `rollout restart` — Flux SSA
strips the `restartedAt` annotation and double-rolls the deployment.

## Security posture

- **Read-only by design.** The bot's RBAC is `get,list` on pods, pods/log,
  nodes, events, PVCs, deployments/statefulsets/daemonsets, ingresses,
  Flux kustomizations/helmreleases/helmrepositories, cert-manager
  certificates, CNPG
  clusters, batch jobs/cronjobs, Longhorn volumes/nodes/settings/
  snapshots/replicas/recurringjobs, k3s
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

### Local mode & the TUI (2026-07-21)

Every command handler is a plain function `def cmd(response_url, args)`
whose only side effect is `post()`. The old exec-and-monkeypatch test
harness that exploited this is now **built into bot.py itself**:

```bash
# extract bot.py from the ConfigMap (or the repo manifest), then:
python3 bot.py nodes
python3 bot.py why monitoring grafana-cnpg-1
python3 bot.py ps mem all
```

With argv present, `use_local_shims()` replaces `k8s_raw`/`promq`/
`am_json`/`post` with kubectl-credential versions (`kubectl get --raw`;
Prometheus/Alertmanager via the apiserver service proxy
`/api/v1/namespaces/monitoring/services/<svc>:<port>/proxy/…`) and runs
one handler, printing the Slack-formatted answer. With no argv it
bootstraps Socket Mode exactly as before — in-cluster behaviour is
unchanged, and slack_bolt is only imported on that path. The shim maps
kubectl error text back onto the `HTTPError` codes the handlers' error
paths expect (`logs prev` → 400 etc.), closing the old harness's one
known gap. `summary`/`backups` are stubbed out in bare local mode (they
need the report ConfigMaps) — the TUI bridges them, see below.

**The TUI — QubeCommander** — `~/git/qube-commander/qube` (own git
repo, README there; renamed 2026-07-22 from `~/git/cluster-bot/tui.py`)
— is an htop-style terminal commander built on the same handlers. It
extracts bot.py + wiki from this repo's manifests (live ConfigMaps as
fallback), renders the Slack mrkdwn answers as colored terminal text,
keeps `ps` as a resident auto-refreshing task view with sort keys,
`/text` row filtering and drill-down, and — its teaching feature — a
live trace pane with two 1:1 views (`F7` flips): the default **how-to
view** translates every call into the command a human would type for
the same answer (idiomatic kubectl with operator greps/filters, the
`backups.velero.io` gotcha, `grafana ▸ explore` steps where only PromQL
can answer), while the **raw view** shows the literal
`kubectl get --raw` calls and decoded PromQL. `summary` and `backups`
run their actual report ConfigMap scripts against a localhost bridge
that rewrites `PROM_BASE`/`API_BASE`/`SLACK_WEBHOOK_URL` onto traced
kubectl calls. Also a one-shot CLI:
`qube [--plain|--raw-trace|--no-trace] <cmd> [args…]`.

Because QubeCommander execs whatever `bot.py`/`HANDLERS` the manifest
defines, new bot commands appear in it automatically — but keep the
module-level names it rewires (`k8s_raw`, `promq`, `am_json`, `post`,
`HANDLERS`, `ALIASES`, `USAGE`, `WIKI_DIR`) stable.
