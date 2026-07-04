# Cluster Slack Bot — on-demand queries via `/cluster`

**Created:** 2026-07-04.

A single-pod Slack bot for read-only cluster queries from Slack. Runs in
**Socket Mode**: the pod opens an outbound WebSocket to Slack and receives
slash commands over it — nothing inbound, the cluster stays LAN-only.
Free-tier Slack feature (no subscription; counts as 1 of the free plan's
10 workspace integrations, alongside the Alertmanager webhook).

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
/cluster nodes                           readiness, kubelet version, CPU load
/cluster logs <ns> <pod> [ctr] [lines]   log tail (pod name PREFIX is enough; default 30, max 200)
/cluster describe <ns> <pod>             pod detail: containers, restarts w/ last exit, its events
/cluster events [ns]                     recent Warning events, newest first
/cluster top [ns]                        top-10 CPU / memory pods (Prometheus)
/cluster restarts [ns]                   pods restarted in the last 24h (Prometheus)
/cluster flux                            Kustomizations + HelmReleases ready/suspended + revision
/cluster velero [n]                      last n Velero backups: phase, errors, age (default 12)
/cluster certs                           cert-manager expiries, soonest first (red <7d, amber <21d)
/cluster cnpg                            DB clusters: ready, primary, replication lag, last backup
/cluster pvcs [ns]                       volume claims: status, capacity, storageclass
/cluster ingresses                       hostname → service map
/cluster alerts                          currently firing Prometheus alerts
/cluster help                            the list above with kubectl equivalents
```

Singular/plural aliases work (`pod`, `deploy`, `log`, `event`, `pvc`,
`ingress`…). `help` shows each command with the kubectl command you'd
type on a terminal.

Replies go through the slash command's `response_url` → visible **only to
the invoker**, in whatever channel the command was typed. `summary` and
`backups` execute the existing report ConfigMaps
(`cluster-summary-report-script`, `backup-summary-report-script`) as
subprocesses with `SLACK_WEBHOOK_URL` overridden to the `response_url` —
the CronJobs and the bot share one copy of the report code.

## Security posture

- **Read-only by design.** The bot's RBAC is `get,list` on pods, pods/log,
  nodes, events, PVCs, deployments/statefulsets/daemonsets, ingresses,
  Flux kustomizations/helmreleases, cert-manager certificates and CNPG
  clusters, plus reuse of the `backup-summary-report` ClusterRole and the
  velero `cloud-credentials` read Role. No write verbs anywhere — a leaked
  Slack token can look at the cluster, never touch it. Do not add
  restart/delete commands.
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
kubectl -n monitoring rollout restart deploy/cluster-slack-bot   # reconnect
```

- The pod pip-installs `slack-bolt` at start — a PyPI outage delays
  (re)starts but never affects a running bot.
- Slack's `response_url` allows 5 posts within 30 min per invocation —
  each command uses 1–2, so this never binds in practice.
- Token rotation: regenerate in the Slack app UI, reseal, restart the
  deployment.
