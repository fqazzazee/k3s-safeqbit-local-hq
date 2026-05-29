# Uptime Kuma

Status / uptime monitoring for the `safeqbit-local-hq` cluster.
Added 2026-05-29.

- **Namespace:** `uptime-kuma`
- **Hostname:** https://uptime.local.safeqbit.com (admin UI *and* status pages, internal only)
- **Manifests:** `apps/safeqbit-local-hq/uptime-kuma/`
- **Image:** `louislam/uptime-kuma:2.3.2` (pinned; bump deliberately)
- **Storage:** `uptime-kuma-data` 2Gi Longhorn RWO at `/app/data` (SQLite — the *only* home for all monitor/status-page/login config)
- **Backup:** `infrastructure/.../velero-schedule-uptime-kuma.yaml` — weekly to B2, 180d retention
- **Deeper metrics:** Grafana/Prometheus handles resource/latency metrics. Uptime Kuma is intentionally scoped to **uptime only**, checked over public **FQDNs** (not cluster-internal IPs/service DNS).

> Why FQDN-only: monitors hit the same path a user would (DNS → ingress → TLS →
> app), so a green check means the whole chain is healthy. Internal Service IPs
> change on redeploy; FQDNs are stable.

---

## Deploy

Pushed via Flux like every other app — no manual `kubectl apply` needed.

```sh
git add apps/safeqbit-local-hq/uptime-kuma \
        apps/safeqbit-local-hq/kustomization.yaml \
        infrastructure/safeqbit-local-hq/configs/velero-schedule-uptime-kuma.yaml \
        infrastructure/safeqbit-local-hq/configs/kustomization.yaml
git commit -m "Add Uptime Kuma"
git push

# Watch Flux pick it up
flux reconcile kustomization apps --with-source
kubectl -n uptime-kuma get pods,pvc,ingress
# Cert (DNS-01 via Cloudflare) — wait for READY=True
kubectl -n uptime-kuma get certificate uptime-kuma-tls
```

## First run — DO THIS IMMEDIATELY

Uptime Kuma has **no default credentials**: the *first* person to load the page
sets the admin account. On a LAN that's an open door until you claim it.

1. Open https://uptime.local.safeqbit.com the moment the pod is Ready.
2. Create the admin username + a strong password (store it in Vaultwarden).
3. Settings → make sure "Disable Auth" stays **off**.

---

## Monitors to create

For **every** monitor below:

- **Monitor Type:** `HTTP(s)`
- **Heartbeat Interval:** `60` seconds
- **Retries:** `2` (mark DOWN only after 2 consecutive failures — rides out a single ingress blip)
- **Heartbeat Retry Interval:** `30` seconds
- **Ignore TLS/SSL error:** **OFF** (certs are real Let's Encrypt — a TLS failure *should* alert)
- **Upside-Down Mode:** OFF
- Leave **Certificate Expiry** notifications ON (Kuma warns ~14d before expiry — a nice safety net on the cert-manager renewals).

### Group: Applications

| Friendly name | URL | Accepted status codes |
|---|---|---|
| AFFiNE       | https://affine.local.safeqbit.com        | 200-299 |
| Authentik    | https://authentik01.local.safeqbit.com   | 200-299 |
| Immich       | https://immich.local.safeqbit.com        | 200-299 |
| NetBox       | https://netbox.local.safeqbit.com        | 200-399 |
| Passzilla    | https://passzilla.local.safeqbit.com     | 200-299 |
| Photoprism   | https://photoprism.local.safeqbit.com    | 200-299 |
| Vaultwarden  | https://vaultwarden.local.safeqbit.com   | 200-299 |

### Group: Infrastructure

| Friendly name | URL | Accepted status codes |
|---|---|---|
| Grafana   | https://grafana.local.safeqbit.com   | 200-399 |
| Longhorn  | https://longhorn.local.safeqbit.com  | 200-299 |

> **Accepted status codes:** some apps redirect an unauthenticated `/` to a
> login page (302) or return 401/403. If a monitor flaps red even though the
> app is up, widen its accepted codes to `200-399` (and add `401,403` if it
> returns those). NetBox/Grafana commonly redirect, hence `200-399` above.
> Tighten back down once you've confirmed the real healthy response.

> **Tip — health endpoints:** a couple of these expose lighter-weight health
> paths you can point the monitor at instead of `/` once it's working:
> Authentik `/-/health/live/`, Immich `/api/server/ping`. Optional.

---

## Core infrastructure monitors (non-FQDN)

The app checks above stay strictly FQDN-based. CoreDNS, etcd, the ingress
controller, and the API server have **no ingress hostname**, so they need
different monitor types pointed at the cluster's *stable* endpoints. These are
not the ephemeral pod IPs that the FQDN-only rule was protecting against — they
are fixed for the life of the cluster:

| Endpoint | Value | Stability |
|---|---|---|
| Ingress VIP (MetalLB) | `10.10.13.50` | Pinned in `metallb-pools.yaml` (reserved-pool) |
| API server | `k3s.local.safeqbit.com:6443` | FQDN, round-robins all 3 nodes |
| kube-dns ClusterIP | `10.43.0.10` (verify) | Fixed Service VIP, not a pod IP |
| Node IPs | `10.10.13.x` (look up) | Static VLAN addresses (10.10.13.0/24) |

Look up the environment-specific ones once:

```sh
kubectl -n kube-system get svc kube-dns                 # -> kube-dns ClusterIP
kubectl get nodes -o wide                               # -> the 3 node INTERNAL-IPs
```

> **Reality check on these checks:** Uptime Kuma can only prove "the port
> answers" / "DNS resolves" / "the health URL says ok". Real quorum, scrape, and
> saturation health come from Prometheus (it already scrapes `etcd`,
> `kubeApiserver`, CoreDNS). Treat these as a fast, at-a-glance liveness layer
> that complements Grafana — not a replacement for it.

### Group: Infrastructure (add to the existing group)

| Friendly name | Type | Target / settings |
|---|---|---|
| **Kubernetes API** | HTTP(s) - Keyword | URL `https://k3s.local.safeqbit.com:6443/readyz`, keyword `ok`, **Ignore TLS error: ON** (API serves its own cert, not Let's Encrypt). `/readyz`, `/livez`, `/healthz` allow anonymous access. |
| **CoreDNS** | DNS | Resolve Server `10.43.0.10`, Port `53`, Resolve Type `A`, Hostname `kubernetes.default.svc.cluster.local` (in-cluster name — tests CoreDNS itself, no upstream dependency). |
| **Ingress-nginx (TCP)** | TCP Port | Host `10.10.13.50`, Port `443`. Isolates "ingress data plane down" from "one app down". |
| **Ingress-nginx (default backend)** | HTTP(s) | URL `http://10.10.13.50`, Accepted codes `404` (unknown Host hits the default backend → proves nginx is routing). **Ignore TLS error: ON** if you use `https`. Optional, complements the TCP check. |
| **etcd — node 1** | TCP Port | Host `<node1-ip>`, Port `2379`. |
| **etcd — node 2** | TCP Port | Host `<node2-ip>`, Port `2379`. |
| **etcd — node 3** | TCP Port | Host `<node3-ip>`, Port `2379`. One monitor per node so a single-node failure (quorum intact) is visibly distinct from two (quorum lost). Port-open only — etcd needs mTLS, so Kuma can't query health; Grafana covers that. |
| **Node reachability** (×3) | Ping | Host `<nodeN-ip>`. ICMP liveness for each Proxmox-hosted k3s VM. Combine with the per-node etcd checks to localize a node outage. |

Use the same interval/retry settings as the app monitors (60s / 2 retries / 30s).

### Other monitor types worth knowing

Uptime Kuma's toolbox goes well beyond HTTP — useful ones for this cluster:

- **Push (passive heartbeat)** — *the* gap-filler vs. Grafana. Create a Push
  monitor, copy its `/api/push/<token>` URL, and append a `curl -fsS <url>` to
  the end of a job. If the job stops running, Kuma alerts on the missed
  heartbeat. Ideal for confirming the **CNPG ScheduledBackups**, **Velero
  schedules**, and the **cnpg-backup-retention CronJob** actually run — "did the
  backup happen?" is something uptime polling can't answer.
- **HTTP(s) - Keyword** — assert the response body contains expected text
  (e.g. a login-page title) so a 200-that-serves-an-error-page still trips.
- **HTTP(s) - JSON query** — query a JSON endpoint and assert a field (e.g.
  Longhorn's API for volume robustness, or an app's `/health` JSON).
- **Postgres / Redis** — real connection checks against the stable
  `*-cnpg-rw.<ns>.svc` / Redis service DNS names using the app secret creds.
  Off by default here (deeper than uptime), but available if you ever want a
  true DB-up signal beyond the app's own FQDN check.
- **TCP Port / DNS / Ping / gRPC** — generic L4/L7 probes as used above.

---

## Notifications (recommended)

You already alert to Slack via Alertmanager. Mirror DOWN/UP events into the same
channel so Uptime Kuma and Prometheus alerts land together:

1. Settings → **Notifications** → Add → type **Slack**.
2. Paste the Slack incoming-webhook URL (same one used by
   `infrastructure/.../configs/alertmanager-slack.yaml`, or a dedicated
   `#uptime` webhook).
3. Set it as **Default enabled** so it auto-attaches to new monitors, then
   tick it on each existing monitor.

---

## Status page

1. **Status Pages → New Status Page.**
   - Name: `SafeQbit HQ`
   - Slug: `safeqbit`  → served at https://uptime.local.safeqbit.com/status/safeqbit
2. Add two groups: **Applications** and **Infrastructure**.
3. Drag the matching monitors into each group.
4. Options worth setting:
   - Show "Powered by Uptime Kuma": your call.
   - **Show certificate expiry:** ON.
   - Theme / logo: optional.
5. Save → **Publish**.

To make `/` redirect to this page, set it as the default status page in its
settings (or just bookmark the `/status/safeqbit` URL).

---

## Troubleshooting

- **Monitor red but app loads in browser:** check the accepted status codes
  (redirect to login?). Use the monitor's "important events" + manual `curl`
  from the pod:
  `kubectl -n uptime-kuma exec deploy/uptime-kuma -- curl -sI https://affine.local.safeqbit.com`
- **All FQDN monitors red:** the pod can't resolve `*.local.safeqbit.com`.
  Confirm from inside the pod:
  `kubectl -n uptime-kuma exec deploy/uptime-kuma -- nslookup affine.local.safeqbit.com`
  If it fails, internal DNS isn't returning the ingress/MetalLB IP for these
  names. Quick fixes: add a CoreDNS rewrite/hosts entry for the ingress LB IP,
  or add `hostAliases` to the Deployment pointing the FQDNs at the ingress IP.
- **UI keeps reconnecting / "Disconnected":** the ingress WebSocket timeout.
  The ingress already sets `proxy-read-timeout`/`proxy-send-timeout` to 3600s;
  confirm those annotations survived on the live object.
- **Pod CrashLoop after image bump:** a minor upgrade may have migrated the
  SQLite schema. Roll the image tag back in `03-deployment.yaml`; restore the
  PVC from the Velero `uptime-kuma-weekly` backup if the DB was corrupted.
- **Infra monitors (etcd/ping/CoreDNS) all red but apps green:** the pod can't
  reach the LAN VLAN (`10.10.13.0/24`) from the pod network. Test egress from
  the pod:
  `kubectl -n uptime-kuma exec deploy/uptime-kuma -- nc -zv 10.10.13.50 443`
  and `... -- nslookup -port=53 kubernetes.default.svc.cluster.local 10.43.0.10`.
  If blocked, check for a NetworkPolicy or that pod→node-VLAN routing is allowed.
- **Ping monitors stuck "Pending"/error:** ICMP from the container needs the raw
  socket to work; the official image normally allows it. If your nodes block
  ICMP, swap the Ping monitor for a TCP Port check (e.g. node `:6443`) instead.

## Restore (DR)

Everything that defines the monitoring stack is in the `uptime-kuma-data` PVC.
Restore the namespace from the latest Velero backup:

```sh
velero backup get | grep uptime-kuma-weekly
velero restore create --from-backup uptime-kuma-weekly-<TIMESTAMP>
```

Then reconcile Flux so the Deployment/Service/Ingress match Git again.
