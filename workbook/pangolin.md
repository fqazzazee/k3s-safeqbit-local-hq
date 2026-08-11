# Pangolin — identity-aware remote access (HTTP + browser RDP/VNC/SSH)

**Namespace:** `pangolin`
**Dashboard:** `https://pangolin.pg.local.safeqbit.com`
**Everything published:** `*.pg.local.safeqbit.com`
**Path in repo:** `apps/safeqbit-local-hq/pangolin/`
**Upstream:** <https://pangolin.net> · docs <https://docs.pangolin.net>

LAN-only — nothing is forwarded from the WAN.

> Sibling stack: [[guacamole]] covers the same RDP/VNC/SSH need via Apache
> Guacamole. They overlap; keep both only while comparing them.

---

## What it does

Pangolin publishes a private service under a public-looking hostname, checks who
you are before letting you through, and reaches the service over a WireGuard
tunnel that the service's own network dials **outbound**. Nothing on the target
side has to accept an inbound connection.

Four programs, each with one job:

| | Job |
|---|---|
| **Pangolin** | The brain and the UI. Holds orgs, users, sites, resources and access policies in Postgres, and *generates Traefik routing* for every resource you publish. |
| **Traefik** | The front door. Terminates TLS, matches the request's hostname to a resource, and asks Badger whether this visitor is allowed. |
| **Badger** | A Traefik plugin, Pangolin's authorization check. It is why Traefik cannot be swapped for another proxy. |
| **Gerbil** | The WireGuard server. Hands out peer configuration and relays traffic between connectors and clients. |
| **Newt** | The connector. Runs *next to your service*, dials Gerbil outbound, and forwards the tunnel's traffic to the local target. Userspace WireGuard, so it needs no privileges. |

Publishing a resource is therefore: create it in the UI → Pangolin writes a
Traefik router → Traefik serves the hostname → Badger authorizes → Gerbil relays
to whichever Newt owns that site.

---

## Shape of the deployment

Upstream's compose runs Traefik with **`network_mode: service:gerbil`**: Traefik
lives inside Gerbil's network namespace, so HTTP (`:80/:443`) and WireGuard
(`:51820/:21820` udp) are **one endpoint**. The Kubernetes equivalent is **two
containers in one pod** — same shared netns — fronted by a single mixed TCP/UDP
LoadBalancer. Keeping that shape is what keeps the rest simple.

It also matters for two concrete reasons:

- MetalLB will not share one IP between two Services unless **both** use
  `externalTrafficPolicy: Cluster`, and `Cluster` is fatal here: its SNAT makes
  each site's WireGuard source port roam about once a minute, so Gerbil's
  hole-punch never registers and no resource router is ever published.
- Newt's exit-node preflight is a plain `GET base_endpoint/ping`. Gerbil does not
  answer that; a Traefik on the same address does.

Mixed TCP/UDP on one LoadBalancer is GA (`MixedProtocolLBService`, k8s ≥1.26);
this cluster runs 1.35 and MetalLB L2 handles it.

**No Helm chart.** The official chart is `0.1.0-alpha.1` with appVersion 1.18.2
against an image at 1.21.1. Everything here is hand-authored plain manifests.

| Component | Image | File | Exposure |
|---|---|---|---|
| Pangolin | `fosrl/pangolin:ee-postgresql-1.21.1` | `08-pangolin-deployment.yaml` | ClusterIP `pangolin` :3000/3001/3002/3003 |
| Gerbil | `fosrl/gerbil:1.4.3` | `11-edge-deployment.yaml` (container 1) | UDP 51820/21820 + TCP 3004 internal |
| Traefik | `traefik:v3.7.9` | `11-edge-deployment.yaml` (container 2, **same pod**) | TCP 80/443 + UDP 443 (h3) |
| Database | CNPG `pangolin-cnpg` | `02-cnpg-cluster.yaml` | Longhorn 5Gi, `instances: 1` |
| Newt (in-cluster) | `fosrl/newt:1.15.0` | `14-newt-deployment.yaml` | none — outbound only |

Traefik loads routing the canonical Pangolin way: HTTP provider →
`http://pangolin:3001/api/v1/traefik-config`, plus a file provider for the four
dashboard routers, plus Badger on every router. No Traefik CRDs, no RBAC.

### Addresses

| What | Address | Notes |
|---|---|---|
| Edge LoadBalancer | **10.10.13.52** | reserved pool `10.10.13.50–59`, `autoAssign:false`, ETP `Local` |
| Edge ClusterIP | **10.43.0.100** (pinned) | pinned because `hostAliases` needs a literal IP |
| ingress-nginx | 10.10.13.50 | the other 16 apps — untouched by this |
| home-assistant-lan | 10.10.13.51 | **do not reuse** — Shelly sensors have it baked in |

`10.43.0.100` sits in the service CIDR's low/static band (Kubernetes allocates
dynamic ClusterIPs from the upper band), so it will not collide.

### DNS

Two records, and only two, however many resources get published:

| Record | Type | Value | Serves |
|---|---|---|---|
| `pg.local.safeqbit.com` | A | `10.10.13.52` | the WireGuard endpoint (`gerbil.base_endpoint`) |
| `*.pg.local.safeqbit.com` | A | `10.10.13.52` | the dashboard and every published resource |

A new resource needs **no DNS change** — the wildcard already answers for it.

This is the same anchor-plus-wildcard shape the rest of the cluster uses with
`ingress.local.safeqbit.com`; Pangolin simply owns its own anchor, because
WireGuard is UDP and `ingress-nginx` publishes only TCP 80/443.

---

## Certificates — one wildcard, issued once

The edge holds a **single** Let's Encrypt certificate for
`pg.local.safeqbit.com` + `*.pg.local.safeqbit.com`, and every router uses it.

- Traefik requests it at startup, from `entryPoints.websecure.http.tls.domains`
  in `10-edge-config.yaml`, and renews it itself at 30 days remaining.
- `prefer_wildcard_cert: true` in `06-pangolin-config.yaml` makes every router
  Pangolin generates ask for that same wildcard rather than a certificate of its
  own. Since Traefik already holds it, **publishing a resource involves no ACME
  round trip** — it is live as soon as the router appears.
- The apex is carried as an explicit SAN. A wildcard covers `*.pg.local...` but
  not `pg.local...` itself, and the apex is the WireGuard endpoint that Newt
  preflights over HTTP.

Wildcards can only be issued over **DNS-01**, which is required here anyway:
these names resolve to private IPs, so Let's Encrypt can never reach an HTTP-01
challenge. The challenge runs against Cloudflare using a token sealed into the
namespace.

> **A wildcard covers one label.** `wiki.pg.local.safeqbit.com` is covered;
> `wiki.docs.pg.local.safeqbit.com` is not. Keep resource names single-label.

Pangolin re-applies the `domains` block of `config.yml` to its database on every
boot (`copyInDomains`), so the config file stays the source of truth and UI edits
to a config-managed domain do not stick.

---

## In-cluster Newt

A pod that dials its own cluster's LoadBalancer IP takes a bad path: it sends to
what looks like an external address, kube-proxy DNATs it back to a local Service,
and the reply returns by a route conntrack cannot reconcile — compounded by
`externalTrafficPolicy: Local`, which the edge requires for WireGuard to see real
source addresses. The tunnel appears to come up and then carries no data.

So the in-cluster connector never uses `10.10.13.52`. `hostAliases` point both
names at the edge's ClusterIP, making it ordinary pod-to-Service traffic:

```
                     pg.local.safeqbit.com
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
time** — `util.ResolveDomain()` → `net.LookupIP()` on a `CGO_ENABLED=0` binary,
which consults `/etc/hosts` first. Pangolin hands out a *name*, not an address.

The same entry covers `pangolin.pg.local.safeqbit.com`, so the control channel
and WebSocket reach Traefik on the ClusterIP too — and TLS still validates,
because routing is by `Host` header and the wildcard covers that name.

Notes:

- If `10.43.0.100` changes, `14-newt-deployment.yaml` must change with it.
- Resource targets from this connector can be plain cluster DNS, e.g.
  `http://vaultwarden.vaultwarden.svc.cluster.local:80`.
- **Fallback:** publish cluster apps from the Docker-host Newt instead, targeting
  their normal `*.local.safeqbit.com` names through ingress-nginx. No in-cluster
  WireGuard at all.

---

## Mail — SMTP2GO relay

Pangolin sends user invites, email verification and password resets. It relays
through **SMTP2GO** rather than talking to recipient mail servers itself, so the
cluster never runs a mail server and never needs port 25 open in either
direction.

| Setting | Value | Where |
|---|---|---|
| `smtp_host` | `mail.smtp2go.com` | `06-pangolin-config.yaml` |
| `smtp_port` | `443` | `06-pangolin-config.yaml` |
| `smtp_secure` | `true` | `06-pangolin-config.yaml` |
| `no_reply` | `pg@safeqbit.com` | `06-pangolin-config.yaml` |
| `smtp_user` / `smtp_pass` | env, sealed | `16-sealed-secret-smtp.yaml` |

- **443 is an SSL port, not a STARTTLS one.** SMTP2GO offers TLS/plain on
  25/80/587/2525/8025 and SSL on 465/8465/**443**. SSL means TLS from the first
  byte, which is exactly nodemailer's `secure: true`, so `smtp_secure: true` is
  required here. On 587 or 2525 it would have to be `false`. It is chosen
  because outbound 443 is the one port nothing filters.
- **Both credentials or neither.** `server/emails/index.ts` builds the transport
  with `auth: smtp_user && smtp_pass ? {...} : null`; one alone gives an
  unauthenticated transport that the relay rejects.
- **Removing the `email:` block disables mail silently** apart from one log line:
  `Email SMTP configuration is missing. Emails will not be sent.`
- `require_email_verification` stays `false`. Accounts arrive by invite or
  through Authentik OIDC, so a verification round trip only adds a failure mode.
- Verify without sending anything:
  `python3 -c 'import smtplib,ssl;s=smtplib.SMTP_SSL("mail.smtp2go.com",443,timeout=20);s.login("USER","PASS");print("ok");s.quit()'`
- `config.yml` is a subPath mount, so restart the Deployment after changing any
  of this. Restart by **deleting the pod**, not `rollout restart`.

---

## Enterprise license (free for homelab)

Browser **RDP, VNC and SSH are Enterprise-Edition features**, gated behind a
license key even when self-hosted. The image here is the EE build; those resource
types stay **locked** until a key is entered.

The key is **free** under **$100k USD gross annual revenue**: create an account
at `app.pangolin.net` → create an organization → **Licenses** → free license
application → paste the key at `/admin/license` after first login.

Misrepresenting revenue to claim the free tier violates the license.

EE ↔ community is a **tag swap, not a one-way door** — the schema is identical,
so `postgresql-1.21.x` works if EE is ever dropped.

---

## Deploy and cutover checklist

1. Merge, let Flux apply, and confirm the edge is up:
   `kubectl -n pangolin get pods` shows `pangolin-edge` **2/2**, and
   `curl -I http://10.10.13.52/ping` returns 200.
2. Watch the wildcard get issued — allow **≥2 min**, `delayBeforeChecks: 120s` is
   deliberate:
   `kubectl -n pangolin logs deploy/pangolin-edge -c traefik | grep -i acme`
3. Verify it before touching DNS, using a hosts override:
   `curl -sv --resolve pangolin.pg.local.safeqbit.com:443:10.10.13.52 https://pangolin.pg.local.safeqbit.com/ 2>&1 | grep -E 'subject|SAN|expire'`
4. **Cut DNS over** in Cloudflare (`safeqbit.com` zone): add
   `pg.local.safeqbit.com` → A `10.10.13.52`, and repoint
   `*.pg.local.safeqbit.com` from the Docker host to `10.10.13.52`. Flip it in one
   edit — two live Pangolins round-robining would be confusing.
5. **First login** at the dashboard; create the initial server admin.
6. **Apply the EE license** at `/admin/license`; confirm RDP/VNC/SSH resource
   types appear.
7. **Authentik OIDC SSO** (runtime state, not a manifest value):
   - In Authentik: an OAuth2/OIDC Provider + Application. Pangolin shows the
     redirect URI when you add the IdP (typically
     `https://pangolin.pg.local.safeqbit.com/auth/idp/<id>/oidc/callback`).
   - In Pangolin → Server Admin → Identity Providers: issuer
     `https://authentik01.local.safeqbit.com/application/o/<slug>/`, client
     id/secret, scopes `openid profile email`.
8. **Create the in-cluster Site** (type: Newt), seal its credentials, and enable
   the connector — the `kubeseal` command is in the header of
   `14-newt-deployment.yaml`. Uncomment `14`/`15` in `kustomization.yaml`.
9. **Other Newt connectors:** the Docker host, and any LAN hosts with
   line-of-sight to RDP/VNC/SSH targets. They need a route to `10.10.13.0/24`.
10. **Decommission the Docker/Portainer stack** once soaked.

> Upstream's `USERS_SERVERADMIN_EMAIL` / `_PASSWORD` bootstrap env vars are
> **deprecated in 1.21** — the app logs a warning telling you to remove them.
> Create the first admin through the UI.

---

## Secrets

Nothing secret is in Git or in a ConfigMap. Two `config.yml` keys are
deliberately absent and injected as env vars instead — both are upstream
fallbacks, not workarounds:

- `server.secret` → **`SERVER_SECRET`** (`readConfigFile.ts` assigns it when the
  key is missing).
- `postgres.connection_string` → **`POSTGRES_CONNECTION_STRING`**
  (`db/pg/driver.ts` checks the env var *before* the config file), fed straight
  from CNPG's `pangolin-cnpg-app` Secret, key `uri`.
- `email.smtp_user` / `email.smtp_pass` → **`EMAIL_SMTP_USER`** /
  **`EMAIL_SMTP_PASS`** (both declared
  `.optional().transform(getEnvOrYaml("EMAIL_SMTP_*"))`, and `getEnvOrYaml`
  returns `process.env[envVar] ?? valFromYaml`, so the environment wins).

| SealedSecret | Keys | For |
|---|---|---|
| `pangolin-server-secret` | `SERVER_SECRET` | session signing + encryption of stored secrets. **Never rotate** — it invalidates every session and breaks stored encrypted values. |
| `pangolin-traefik-cloudflare` | `email`, `dnsApiToken`, `zoneApiToken` | DNS-01. The cert-manager Cloudflare token re-sealed for this namespace, same zone scope. |
| `pangolin-newt` | `NEWT_ID`, `NEWT_SECRET` | the in-cluster connector, issued by the UI. |
| `pangolin-smtp` | `EMAIL_SMTP_USER`, `EMAIL_SMTP_PASS` | the SMTP2GO relay login. Cheap to rotate — it is an SMTP2GO user, unrelated to Pangolin state. |

DB password: CNPG creates `pangolin-cnpg-app` itself. Reseal commands are in the
header comments of `04-` and `05-`.

---

## Backups

- **CNPG ScheduledBackup** `pangolin-cnpg-backup` — weekly Sun 03:30 UTC,
  volumeSnapshot (`longhorn-velero`). This is the important one: orgs, users,
  sites, resources, policies, license registration. Retention via
  `configs/cnpg-backup-retention.yaml`.
- **Velero** `pangolin-bimonthly` — 11th & 26th 04:15 UTC → B2, ttl 28d (keep
  last 2). Catches Gerbil's key PVC, the ACME state and the k8s objects. See
  [[backup-strategy]].

Losing Gerbil's key PVC is not fatal, but forces every Newt/Olm peer to
re-handshake against a new server key.

---

## Gotchas

- **Gerbil must track Pangolin's release.** Nothing below **1.4.2** is safe here:
  it fixed *"cache timeout of 2.5s to record hp; fixes registering issue when
  endpoint was the same"*, and every exit node in this deployment shares one
  endpoint. The failure is indirect — the Newt connects, the server logs `Site
  last hole punch is too old; skipping this register`, `Config version` stays
  `0`, and **no resource router is published**, so resources 404. Pinned at
  **1.4.3**.
- **`externalTrafficPolicy: Local` is required, not preferred** — see above.
- **DNS-01 needs both propagation settings.** `disableANSChecks: true` because
  lego's default check queries Cloudflare's *authoritative* nameservers on :53,
  which pods cannot reach; and `delayBeforeChecks: 120s` because with that check
  disabled lego stops waiting and asks Let's Encrypt within ~13s, before the TXT
  record has propagated to LE's resolvers. Dropping either breaks issuance.
- **`config.yml` is a subPath mount**, so it does not live-update. After editing
  `06-pangolin-config.yaml`, restart the Deployment.
- **Restart by deleting the pod**, not `kubectl rollout restart` — Flux's SSA
  strips `restartedAt` and you get a double roll.
- **The ACME PVC's name is load-bearing.** The `nfs-truenas` StorageClass sets
  `pathPattern: ${.PVC.namespace}-${.PVC.name}-${.PV.name}`, but `${.PV.name}` is
  not a token nfs-subdir-external-provisioner understands, so it stays literal
  and the real directory is just `<namespace>-<pvc name>`. A PVC of a given name
  always re-adopts what a previous PVC of that name left behind — deleting and
  recreating it does *not* give you clean state; renaming it does. This affects
  every `nfs-truenas` volume in the cluster (netbox, authentik, grafana,
  guacamole all carry the literal suffix).
- **Longhorn RWX was considered and skipped** for the ACME volume. It works, but
  only by standing up a share-manager NFS pod, and no volume in this cluster does
  that today — the edge's ability to start is not where to introduce it.

See also: [[cnpg-strategy]], [[guacamole]], [[backup-strategy]],
[[node-loss-resilience]], [[dns-search-amplification]].
