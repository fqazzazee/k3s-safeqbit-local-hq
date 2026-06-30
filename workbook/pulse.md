# Pulse

Single-pane-of-glass monitoring for the `safeqbit-local-hq` estate: the
**Proxmox hosts**, this **k3s cluster**, and the standalone **Docker host** — all
in one dashboard. ([rcourtman/Pulse](https://github.com/rcourtman/Pulse))

Server added 2026-06-29. k3s agent added 2026-06-30.

- **Namespace:** `pulse`
- **Hostname:** https://pulse.local.safeqbit.com (admin UI, internal only)
- **Manifests:** `apps/safeqbit-local-hq/pulse/`
- **Image:** `rcourtman/pulse:v5.1.35` (pinned; bump deliberately after reading upstream release notes)
- **Server port:** `7655` (ClusterIP `pulse`, in-cluster DNS `pulse.pulse.svc.cluster.local:7655`)
- **Storage:** `pulse-data` 2Gi Longhorn RWO at `/data` — holds config, the **encrypted target credentials** (Proxmox tokens, agent tokens), discovered nodes, alert config, and history. **The only home for that config** (targets are added in the UI, not in Git).
- **Web-UI auth:** built-in, `PULSE_AUTH_USER` / `PULSE_AUTH_PASS` from SealedSecret `pulse-auth` (`03-sealed-secret.yaml`). Plaintext pass auto-hashed on startup → auth enforced from first boot, no open window. Admin password stored in Vaultwarden.
- **Backup:** `infrastructure/.../velero-schedule-pulse.yaml` — weekly to B2, Sundays 05:00 UTC, 180d retention.

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

## The k3s agent (DaemonSet)

To make the cluster appear in Pulse, the **unified agent** runs as a **DaemonSet**
(`06-agent-rbac.yaml`, `07-agent-sealed-secret.yaml`, `08-agent-daemonset.yaml`) —
one pod per node, automatically including any node added later.

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
- **`PULSE_AGENT_ID=safeqbit-local-hq` (critical):** all DaemonSet pods share the
  one token and this **shared ID**, so the cluster registers as **one logical
  agent**, not three. Without it each pod invents its own ID and the server
  rejects them with *"API token is already in use by agent …"*.

---

## Which targets need an agent?

| Target | Agent? | How |
|---|---|---|
| **Proxmox hosts** | No (agentless) | Add each PVE host in the UI with a Proxmox **API token**. Optional: install the host agent on a PVE node for temps/SMART. |
| **k3s cluster** | Yes — DaemonSet | One manifest set, one agent pod per node (above). |
| **Docker host** (standalone) | Yes — one agent | Install the unified agent on that box (docker mode); it auto-detects Docker. Generate a token with the docker/host report scope in the UI. |

---

## Deploy

GitOps via Flux like every other app — no manual `kubectl apply`. The server went
straight to `main`; the agent went via PR (`ops/pulse-k3s-agent`, PR #9).

```sh
# Watch Flux pick it up
kubectl -n pulse get pods,pvc,ingress
kubectl -n pulse get certificate pulse-tls          # DNS-01 via Cloudflare — wait READY=True
kubectl -n pulse get ds pulse-agent                 # 3/3 desired/ready
kubectl -n pulse logs -l app.kubernetes.io/component=agent --tail=20
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

---

## Troubleshooting

- **Agent: `API token is already in use by agent "mac-…"`** — `PULSE_AGENT_ID` is
  missing/blank on the DaemonSet. It must be set to a shared value
  (`safeqbit-local-hq`) so all pods report as one agent.
- **Agent log: `Failed to fetch remote config … 403 Forbidden`** — harmless. The
  agent tries to pull an upstream remote config; it falls back to local defaults.
  Not a connection problem with the Pulse server.
- **Agent not showing in UI** — check `kubectl -n pulse logs -l
  app.kubernetes.io/component=agent` for a `PULSE_URL`/token error; confirm the
  pods can reach `pulse.pulse.svc.cluster.local:7655`.
- **Cert `pulse-tls` stuck `READY=False`** — DNS-01 challenge in progress
  (`delayBeforeChecks`); usually issues in 2–5 min. `kubectl -n pulse get
  challenge`.
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
