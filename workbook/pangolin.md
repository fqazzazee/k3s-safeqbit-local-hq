# Pangolin — identity-aware remote access (HTTP + browser RDP/VNC/SSH)

**Created:** 2026-06-15 · **Removed from cluster:** 2026-06-18 · **Redeployed:** 2026-08-10
**Namespace:** `pangolin`
**Dashboard:** `https://pangolin.local.safeqbit.com`
**Path in repo:** `apps/safeqbit-local-hq/pangolin/`
**Upstream:** <https://pangolin.net> · docs <https://docs.pangolin.net>

Pangolin ([fosrl/pangolin](https://github.com/fosrl/pangolin)) is an
identity-based remote-access platform built on WireGuard. Resources (web apps,
**RDP**, **VNC**, **SSH**) are published behind Pangolin's own Traefik + Badger
auth middleware and reached through **Newt** site connectors over WireGuard
tunnels managed by **Gerbil**.

This deployment **replaces the single-host Docker/Portainer stack**. It is
LAN-only — nothing is forwarded from the WAN.

> Sibling stack: [[guacamole]] (`workbook/guacamole.md`) covers the same
> RDP/VNC/SSH remote-access need via Apache Guacamole. They overlap; keep both
> only while comparing them.

---

## The one thing that makes this work on k3s

Upstream's compose runs Traefik with **`network_mode: service:gerbil`** —
Traefik lives inside Gerbil's network namespace, so HTTP (`:80/:443`) and
WireGuard (`:51820/:21820` udp) are **one endpoint**. On the Docker host that
endpoint is a macvlan LAN address.

The June 2026 attempt tried to reproduce that with two Services on two MetalLB
IPs, and every subsequent problem flowed from the split. **The correct
translation is two containers in one pod** — a shared netns is exactly what
compose's `network_mode: service:gerbil` is — fronted by a single mixed TCP/UDP
LoadBalancer.

That collapses three separate June workarounds at once:

| June problem | Why it existed | Now |
|---|---|---|
| Edge and Gerbil needed separate IPs (.51/.52) | MetalLB refuses to share an IP unless **both** Services use `Cluster`, and `Cluster`'s SNAT makes the WireGuard source port roam ~1/min so Gerbil's hole-punch never registers | One pod → one Service → one IP, and `externalTrafficPolicy: Local` is kept |
| `http-echo` `/ping` sidecar on Gerbil | Newt's exit-node preflight is `GET base_endpoint/ping`, which only a front Traefik answers — Gerbil never serves `/ping` | Traefik *is* on that address. (Newt ≥1.15 also skips the preflight entirely when there is a single exit node — `newt/handlers.go:143`) |
| In-cluster Newt couldn't carry data | It dialled the cluster's own MetalLB IP — a hairpin, made worse by `Local` | It never uses the MetalLB IP; see "In-cluster Newt" below |

Mixed TCP/UDP on one LoadBalancer is GA (`MixedProtocolLBService`, k8s ≥1.26);
this cluster runs 1.35 and MetalLB L2 handles it.

---

## Architecture as deployed

**No Helm chart.** The official chart is abandoned — still `0.1.0-alpha.1` on
ghcr with appVersion 1.18.2, while the image is at 1.21.1 — and in June its
Traefik was broken in both standalone and controller mode. Everything here is
hand-authored plain manifests.

| Component | Image | Where | Exposure |
|---|---|---|---|
| **Pangolin** (app/API/dashboard) | `fosrl/pangolin:ee-postgresql-1.21.1` | `08-pangolin-deployment.yaml` | ClusterIP `pangolin` :3000/3001/3002/3003 |
| **Gerbil** (WireGuard) | `fosrl/gerbil:1.4.3` | `11-edge-deployment.yaml`, container 1 | UDP 51820/21820 + TCP 3004 (internal only) |
| **Traefik** (edge proxy) | `traefik:v3.7.9` | `11-edge-deployment.yaml`, container 2 — **same pod, same netns** | TCP 80/443 + UDP 443 (h3) |
| **Database** | CNPG `pangolin-cnpg` | `02-cnpg-cluster.yaml` | Longhorn 5Gi, `instances: 1` |
| **Newt** (in-cluster connector) | `fosrl/newt:1.15.0` | `14-newt-deployment.yaml` | none — outbound only |

Traefik loads routing the canonical Pangolin way: HTTP provider →
`http://pangolin:3001/api/v1/traefik-config` (Pangolin generates a router per
published resource) + a file provider for the dashboard routers + the **Badger**
plugin on every router. No Traefik CRDs, no Kubernetes RBAC.

### Addresses

| What | Address | Notes |
|---|---|---|
| Edge LoadBalancer | **10.10.13.52** | reserved pool `10.10.13.50–59`, `autoAssign:false`, ETP `Local` |
| Edge ClusterIP | **10.43.0.100** (pinned) | pinned because `hostAliases` needs a literal IP |
| ingress-nginx | 10.10.13.50 | existing |
| home-assistant-lan | 10.10.13.51 | **do not reuse** — Shelly sensors have it baked in |

`10.43.0.100` sits in the service CIDR's low/static band (Kubernetes allocates
dynamic ClusterIPs from the upper band), so it will not collide.

### Secrets are env vars, not config-file text

Two keys are deliberately absent from the `config.yml` ConfigMap and injected as
env vars, so nothing secret is in Git. Both fallbacks are upstream behaviour:

- `server.secret` → **`SERVER_SECRET`** — `readConfigFile.ts` assigns
  `data.server.secret = process.env.SERVER_SECRET` when the key is missing.
- `postgres.connection_string` → **`POSTGRES_CONNECTION_STRING`** —
  `db/pg/driver.ts` checks the env var *before* the config file. Fed straight
  from CNPG's auto-generated `pangolin-cnpg-app` Secret, key `uri`.

> Upstream's `USERS_SERVERADMIN_EMAIL` / `_PASSWORD` bootstrap env vars are
> **deprecated in 1.21** (the app logs a warning telling you to remove them).
> Create the first admin through the UI.

---

## In-cluster Newt — the hairpin, and why it is fixed

This is the failure that killed the June deployment, so it is worth stating
precisely.

A pod that dials its own cluster's MetalLB IP hairpins: it sends to what looks
like an external address, kube-proxy DNATs it back to a local Service, and the
reply returns by a path conntrack does not reconcile — compounded by
`externalTrafficPolicy: Local`, which the edge requires for WireGuard to work
at all. The tunnel appears to come up and then carries no data.

**The fix is to never hairpin.** `gerbil.base_endpoint` is a **hostname**
(`pangolin-edge.local.safeqbit.com`), not an IP — which is also upstream's own
default, so this is a supported shape rather than a trick — and the in-cluster
Newt overrides it with `hostAliases`:

```
                     pangolin-edge.local.safeqbit.com
                                  │
        ┌─────────────────────────┴─────────────────────────┐
   LAN / Docker-host Newt                          in-cluster Newt
   resolves via Cloudflare                    resolves via /etc/hosts
             ↓                                          ↓
      10.10.13.52  (MetalLB, ETP Local)         10.43.0.100  (ClusterIP)
             └──────────────► edge pod ◄────────────────┘
                       (gerbil + traefik, one netns)
```

This works because **Newt resolves the WireGuard endpoint client-side at connect
time**: `util.ResolveDomain()` → `net.LookupIP()` on a `CGO_ENABLED=0` Go
binary, which consults `/etc/hosts` first. The server hands out a *name*, not a
resolved address.

The same `hostAliases` entry also covers `pangolin.local.safeqbit.com`, so the
connector's control channel and WebSocket reach Traefik on the ClusterIP too —
and TLS still validates, because routing is by `Host` header and the cert for
that name is real.

Notes:

- Newt needs **no privileges**: it is userspace WireGuard (netstack). Only
  Gerbil needs `NET_ADMIN`.
- If `10.43.0.100` ever changes, `14-newt-deployment.yaml` must change with it.
- Resource targets from the in-cluster Newt can be plain cluster DNS, e.g.
  `http://vaultwarden.vaultwarden.svc.cluster.local:80`.
- **Fallback if this still misbehaves:** publish cluster apps from the
  Docker-host Newt instead, targeting their normal
  `*.local.safeqbit.com` names through ingress-nginx (10.10.13.50). No
  in-cluster WireGuard involved at all.

---

## ⚠️ RDP/VNC/SSH requires an Enterprise license key (free for homelab)

Browser **RDP, VNC and SSH are Enterprise-Edition features**, gated behind a
license key even when self-hosted. The image here is the EE build; those
resource types stay **locked** until a key is entered.

The key is **free** for individuals/orgs under **$100k USD gross annual
revenue** (homelab qualifies): create an account at **app.pangolin.net** → create
an organization → **Licenses** → free license application → paste the key at
`https://pangolin.local.safeqbit.com/admin/license` after first login.

Misrepresenting revenue to claim the free tier violates the license.

EE ↔ community is a **tag swap, not a one-way door** — the schema is identical,
so `postgresql-1.21.x` (community) works if EE is ever dropped.

---

## ✅ Post-deploy checklist

### Phase 1 — before first login

1. **DNS (Cloudflare, `safeqbit.com` zone).** The edge is a different IP from
   ingress-nginx, so it gets its own anchor:
   - `pangolin-edge.local.safeqbit.com` → **A `10.10.13.52`** (the anchor; this
     is also `gerbil.base_endpoint`, so change it here if the IP ever moves)
   - `pangolin.local.safeqbit.com` → **CNAME** `pangolin-edge.local.safeqbit.com`
   - **each published resource** → **CNAME** `pangolin-edge.local.safeqbit.com`
     (one per resource — the cluster uses no wildcard). ⚠️ resource names share
     the `local.safeqbit.com` namespace with the nginx apps; don't reuse a host.
2. Confirm the edge came up: `kubectl -n pangolin get svc pangolin-edge-lan`
   shows `10.10.13.52`, and `curl -I http://10.10.13.52/ping` returns 200.
3. First cert issue takes ≥2 min — `delayBeforeChecks: 120s` is deliberate
   (see gotchas).

### Phase 2 — in the UI

4. **First login:** browse to the dashboard, create the initial server admin.
5. **EE license:** apply the free key at `/admin/license`. Verify the RDP/VNC/SSH
   resource types appear.
6. **Authentik OIDC SSO** (runtime/DB state, not a manifest value):
   - In **Authentik**: create an OAuth2/OIDC Provider + Application. The
     redirect URI is shown by Pangolin when you add the IdP (typically
     `https://pangolin.local.safeqbit.com/auth/idp/<id>/oidc/callback`).
   - In **Pangolin → Server Admin → Identity Providers**: issuer
     `https://authentik01.local.safeqbit.com/application/o/<slug>/`, client
     id/secret, scopes `openid profile email`.
7. **Create the in-cluster Site** (type: Newt), then seal its credentials and
   enable the connector — the exact `kubeseal` command is in the header of
   `14-newt-deployment.yaml`. Uncomment `14`/`15` in `kustomization.yaml`.
8. **Other Newt connectors:** Docker host and any LAN hosts with line-of-sight
   to RDP/VNC/SSH targets. They need to route to `10.10.13.0/24`.
9. **Decommission the Docker/Portainer stack** once soaked.

---

## Secrets

- `pangolin-server-secret` (SealedSecret, key `SERVER_SECRET`) — pre-generated
  so it never rotates; rotating it invalidates every session and breaks stored
  encrypted values.
- `pangolin-traefik-cloudflare` (SealedSecret, keys `email`/`dnsApiToken`/
  `zoneApiToken`) — the cert-manager Cloudflare token re-sealed for this
  namespace, for DNS-01. Same scope (zone `safeqbit.com`).
- `pangolin-newt` (SealedSecret, `NEWT_ID`/`NEWT_SECRET`) — phase 2 only.
- DB password: CNPG auto-creates `pangolin-cnpg-app`; nothing to seal.

Reseal commands are in the header comments of `04-` and `05-`.

---

## Backups

- **CNPG ScheduledBackup** `pangolin-cnpg-backup`: weekly Sun 03:30 UTC,
  volumeSnapshot (`longhorn-velero`). This is the DB — orgs, users, sites,
  resources, policies, license registration. Retention via
  `configs/cnpg-backup-retention.yaml`.
- **Velero** `pangolin-bimonthly`: 11th & 26th 04:15 UTC → B2, ttl 28d (keep
  last 2). Catches Gerbil's WG key PVC, the Traefik ACME state, and k8s objects.
  See [[backup-strategy]].

Losing Gerbil's key PVC is not fatal but forces every Newt/Olm peer to
re-handshake against a new server key.

---

## Decisions & gotchas

- **Gerbil version must track Pangolin.** Pangolin's own build pairs each
  release with the *latest* Gerbil. In June the chart pinned Gerbil **1.3.1**
  against Pangolin 1.19.2 and it had a hole-punch *registering* bug: the Newt
  connects but the server logs `Site last hole punch is too old; skipping this
  register`, `Config version` stays `0`, the Newt times out on
  `newt/wg/get-config`, and **no resource router is published** (resources 404,
  cert stuck "pending"). Gerbil 1.4.2's changelog — *"Add cache timeout of 2.5s
  to record hp; fixes registering issue when endpoint was the same"* — is
  exactly that case, since all exit nodes share one endpoint. That cost a long
  debug chasing firewall/hairpin/MetalLB. Pinned here at **1.4.3**.
- **`externalTrafficPolicy: Local` is required, not preferred.** `Cluster`'s
  SNAT rewrites the site's source to a pod IP with a port that roams every
  ~minute, so the WG endpoint flaps and hole-punch (UDP 21820) can't learn the
  site's real address. Symptom is a 404 + "pending" cert, nowhere near the cause.
- **DNS-01, with two non-default settings** (`10-edge-config.yaml`). HTTP-01
  cannot work — the domain resolves only to private IPs. And lego needs:
  `disableANSChecks: true`, because its default propagation check queries the
  zone's *authoritative* Cloudflare NS on :53 directly, which pods here cannot
  reach (`did not return the expected TXT`); and `delayBeforeChecks: "120s"`,
  because with the authoritative check off lego stopped waiting and asked LE
  within ~13s, so LE saw "No TXT record found" before the record propagated.
  `resolvers: 1.1.1.1/1.0.0.1` back the recursive check. Do not drop either.
- **ACME state lives on an RWX PVC** (`pangolin-acme`, nfs-truenas): the edge
  mounts it RW at `/letsencrypt`, the **server** mounts it RO at
  `/app/config/letsencrypt`. Pangolin's EE build scrapes Traefik's `acme.json`
  to flip a resource's cert status pending → valid; with an RWO volume the
  server warned `cannot stat config/letsencrypt/acme.json` every 5s and every
  resource cert was stuck "pending". A resource only goes valid once the edge
  *actually issues* its cert (lazy ACME on first request).
- **`net.ipv4.ip_forward` is set by an init container.** A fresh pod netns
  starts with forwarding off. Traffic Traefik originates works regardless, but a
  Client reaching a Site is *forwarded* traffic and would silently blackhole
  while the tunnel looks healthy.
- **PSA `privileged`** on the namespace: Gerbil needs `NET_ADMIN`, Traefik binds
  :80/:443 as root, the init container writes a sysctl.
- **No NetworkPolicies.** The rest of the cluster runs without per-app policies;
  k3s enforces them when present, and Pangolin needs egress to Authentik (OIDC),
  app.pangolin.net (license), GitHub (badger plugin fetch) and Let's Encrypt.
- **`dnsConfig` sets `ndots:1`** on every pod here — pods inherit search domain
  `local.safeqbit.com`, and the default `ndots:5` makes short FQDNs get the
  search domain appended first. See [[project-dns-search-amplification]].
- **ConfigMap changes need a restart.** `config.yml` is a `subPath` mount, which
  does not live-update.
- **Badger version** tracks Pangolin's installer, which builds against the
  *latest* badger tag. Bump `v1.5.0` in `10-edge-config.yaml` when upgrading.
- **Telemetry and update notifications off**, matching the cluster's privacy
  posture (cf. Authentik error reporting).
- TrueNAS still has an orphaned `pangolin-acme` subdir from the June deployment
  (old `acme.json`/TLS keys) — delete it on the NAS if a stale cert appears.

See also: [[cnpg-strategy]] (DB instance-count rationale), [[guacamole]]
(parallel RDP/VNC stack), [[backup-strategy]], [[node-loss-resilience]].
