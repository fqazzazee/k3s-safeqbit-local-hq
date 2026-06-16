# Pangolin — identity-aware remote access (HTTP + browser RDP/VNC/SSH)

**Created:** 2026-06-15
**Namespace:** `pangolin`
**Dashboard:** `https://pangolin.local.safeqbit.com`
**Path in repo:** `apps/safeqbit-local-hq/pangolin/`
**Upstream:** <https://pangolin.net> · docs <https://docs.pangolin.net>

Pangolin ([fosrl/pangolin](https://github.com/fosrl/pangolin)) is an
identity-based remote-access platform built on WireGuard. Resources (web apps,
**RDP**, **VNC**, **SSH**) are published behind Pangolin's own Traefik + Badger
auth middleware and reached through **Newt** site connectors over WireGuard
tunnels managed by **Gerbil**. Deployed here for internal use, primarily for the
1.19 **browser RDP/VNC/SSH** feature.

> Sibling stack: [[guacamole]] (`workbook/guacamole.md`) covers the same
> RDP/VNC/SSH remote-access need via Apache Guacamole. Pangolin and Guacamole
> overlap; keep both only while comparing them.

---

## ⚠️ RDP/VNC requires an Enterprise license key (free for homelab)

Browser **RDP, VNC and SSH are Enterprise-Edition features** — they are gated
behind a license key even when self-hosted. The image deployed here is the EE
build (`fosrl/pangolin:ee-postgresql-1.19.2`); RDP/VNC stay **locked** until a
key is entered.

The key is **free** for individuals / orgs under **$100k USD gross annual
revenue** (homelab qualifies). To get and apply it:

1. Create an account at **app.pangolin.net** → create an **organization**.
2. **Licenses** section → complete the free **license application**.
3. Paste the key at **`https://pangolin.local.safeqbit.com/admin/license`**
   (Server Admin) after first login.

Misrepresenting revenue to claim the free tier violates the license. Genuine
homelab use is eligible.

---

## Architecture as deployed

Topology: the Helm chart deploys **Pangolin + Gerbil + DB** only. The chart's
own Traefik is **disabled** (it's broken/absent in 0.1.0-alpha.x — see Decisions
below), so the edge is a **hand-authored dedicated Traefik** wired the canonical
Pangolin way (HTTP provider + file provider + Badger plugin). The chart runs in
`deployment.type=standalone, mode=multi` purely to get the app+Gerbil workloads;
`traefik.enabled=false`, `controller.enabled=false`.

| Component | Image | How it's deployed | Exposure / notes |
|---|---|---|---|
| **Pangolin** (app/API/dashboard) | `fosrl/pangolin:ee-postgresql-1.19.2` | Helm chart | ClusterIP `pangolin` :3000/3001/3002/3003. EE + Postgres + 1.19. Generates Traefik routing at `/api/v1/traefik-config`. |
| **Traefik** (edge proxy `pangolin-edge`) | `traefik:v3.6.15` | **hand-authored** (08–10) | **LoadBalancer `10.10.13.51`**; HTTP provider → `pangolin:3001` + file provider (dashboard routes) + **Badger** plugin (`v1.4.1`); **own ACME**, Cloudflare DNS-01, per-host certs. Does NOT use nginx-ingress/cert-manager. |
| **Gerbil** (WireGuard) | `fosrl/gerbil:1.3.1` | Helm chart | **LoadBalancer `10.10.13.52`** (own IP, `externalTrafficPolicy: Local`), UDP 51820/21820 + TCP 3004 + **TCP 80** (http-echo `/ping` sidecar, added via postRenderer); `NET_ADMIN`; PVC for WG key. See "Newt exit-node /ping" below. |
| **Database** | CNPG `pangolin-cnpg` | hand-authored (02) | Longhorn 5Gi, `instances:1`; CNPG app secret `pangolin-cnpg-app` key `uri`. |

Chart deployed via Flux **HelmRelease** against the official OCI chart
`oci://ghcr.io/fosrl/helm-charts/pangolin:0.1.0-alpha.0` (source `fossorial` in
`infrastructure/.../controllers/sources.yaml`). Chart AppVersion is 1.18.2; the
EE/1.19 image is pinned via `images.pangolin.tag`. The dedicated Traefik
(`08-traefik-config.yaml` ConfigMap, `09-traefik-deployment.yaml`,
`10-traefik-service.yaml`, `07-traefik-acme-pvc.yaml`) is plain manifests in the
same app dir, adapted from `fosrl/pangolin@1.19.2 install/config/traefik/`.

### MetalLB IPs

`10.10.13.50` = ingress-nginx (existing). From the **reserved** pool
(`10.10.13.50–59`, `autoAssign:false`) via `metallb.io/loadBalancerIPs`: the edge
Traefik pins **`10.10.13.51`** and Gerbil pins **`10.10.13.52`**, each its OWN IP,
both **`externalTrafficPolicy: Local`**.

> **Why NOT a shared IP** (tried, reverted): stock Pangolin runs Traefik + Gerbil on
> one endpoint, so co-locating them on `10.10.13.51` via `allow-shared-ip` is tempting.
> But MetalLB **refuses to share an IP unless both services use `Cluster`** ("can't
> change sharing key" → edge loses its IP under `Local`). And `Cluster` is fatal to
> WireGuard: its SNAT rewrites the site's source to a pod IP (`10.42.x`) with a port
> that **roams every ~minute**, so the WG endpoint flaps and Gerbil's hole-punch
> (UDP 21820) can't learn the site's real address (`last hole punch is too old`). The
> flapping tunnel kept the Newt site from staying online, so Pangolin never published
> the RDP resource (404 + cert pending). Conclusion: separate IPs + `Local` is required
> for a stable tunnel; the `/ping` problem is solved with a sidecar instead (below).

### Newt exit-node `/ping` (why Gerbil has an http-echo sidecar)

A Newt site connector, before bringing up WireGuard, runs an HTTP preflight
`GET http://<base_endpoint>:80/ping` to confirm the exit node is reachable. That
`/ping` is answered **only by a front-door Traefik** (Gerbil serves
`/peer`,`/healthz`,… but **never `/ping`** — true even on the latest Gerbil). Since
Gerbil has its own IP (`base_endpoint: 10.10.13.52`) with nothing on :80, the
preflight timed out — Newt logged `Failed to ping exit node (http://10.10.13.52/ping)`
and **never reached the handshake**, so Gerbil sat idle. Fix: a tiny
`hashicorp/http-echo` **sidecar** in the Gerbil pod (binds `:8080`, 200s every path)
exposed as **port 80** on Gerbil's LB (postRenderer patches in `06`). Now
`10.10.13.52:80/ping` → `OK` and `:51820/udp` → Gerbil's WireGuard, on Gerbil's own
`Local` IP. (Newt's *post-handshake* check is a separate ICMP ping over the tunnel to
Gerbil's `100.89.x` IP, allowed by Gerbil's wg0 firewall.) Newt must run on a network
that can **route to `10.10.13.0/24`** (base_endpoint is a private IP). Browser RDP/VNC
also requires **Newt ≥ 1.13.0** (the RDP/SSH gateway runs inside Newt;
`authDaemonMode: native`).

---

## ✅ Post-deploy checklist (manual, not in Git)

1. **DNS (Cloudflare, `safeqbit.com` zone)** — mirror the cluster's CNAME-to-anchor
   pattern (existing apps are `*.local.safeqbit.com` CNAME →
   `ingress.local.safeqbit.com` A → `10.10.13.50`). Pangolin's Traefik is a
   *different* IP, so it gets its **own anchor**:
   - `pangolin-ingress.local.safeqbit.com` → **A `10.10.13.51`** (the Traefik
     LoadBalancer; change here if the IP ever moves)
   - `pangolin.local.safeqbit.com` → **CNAME `pangolin-ingress.local.safeqbit.com`**
     (dashboard)
   - **each published resource** → **CNAME `pangolin-ingress.local.safeqbit.com`**
     (e.g. `winbox.local.safeqbit.com`). One CNAME per resource — no wildcard
     (the cluster uses none). ⚠️ resource names share the `local.safeqbit.com`
     namespace with the nginx apps, so don't reuse an existing host name.
   - Gerbil uses its own IP `10.10.13.52` (no DNS record needed); Newt dials
     `base_endpoint=10.10.13.52` directly.
2. **First login:** browse to the dashboard, create the initial server admin.
3. **EE license:** apply the free key at `/admin/license` (see top section).
   Verify browser RDP/VNC/SSH resource types appear.
4. **Authentik OIDC SSO** (done in-UI, it is DB/runtime state, not a chart value):
   - In **Authentik**: create an **OAuth2/OIDC Provider** + Application; redirect
     URI is shown by Pangolin when you add the IdP (typically
     `https://pangolin.local.safeqbit.com/auth/idp/<id>/oidc/callback`).
   - In **Pangolin → Server Admin → Identity Providers**: add an OIDC provider
     with Authentik's issuer (`https://authentik01.local.safeqbit.com/application/o/<slug>/`),
     client id/secret, scopes `openid profile email`.
   - See <https://docs.pangolin.net> → Identity Providers.
5. **Newt connector(s):** install a Newt site connector (**v1.13.0+** required for
   browser RDP/VNC/SSH and native SSH mode) on a LAN host with line-of-sight to
   the RDP/VNC targets; register it as a Site in the dashboard. Then publish
   RDP/VNC/SSH resources pointing at the target host:port (e.g. Windows `:3389`).

---

## Secrets

- `pangolin-server-secret` (SealedSecret, key `SERVER_SECRET`) — pre-generated so
  the app secret never rotates on chart upgrade (chart would otherwise generate
  one). Referenced via `pangolin.secret.existingSecretName`.
- `pangolin-traefik-cloudflare` (SealedSecret, keys `email`/`dnsApiToken`/
  `zoneApiToken`) — the Cloudflare API token re-sealed for this namespace from
  the existing `cloudflare-api-token` in `cert-manager`. Used by Traefik for
  DNS-01. Same scope as cert-manager's (zone `safeqbit.com`).
  - Reseal command (no plaintext printed):
    ```bash
    CF=$(kubectl get secret cloudflare-api-token -n cert-manager -o jsonpath='{.data.api-token}' | base64 -d)
    kubectl create secret generic pangolin-traefik-cloudflare -n pangolin \
      --from-literal=email=fadi@safeqbit.com \
      --from-literal=dnsApiToken="$CF" --from-literal=zoneApiToken="$CF" \
      --dry-run=client -o yaml \
    | kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system --format yaml \
    > apps/safeqbit-local-hq/pangolin/05-sealed-secret-cloudflare.yaml; unset CF
    ```
- DB password: CNPG auto-creates `pangolin-cnpg-app`; the HelmRelease consumes its
  `uri` key directly. Nothing to seal.

---

## Backups

- **CNPG ScheduledBackup** `pangolin-cnpg-backup`: weekly Sun 03:30 UTC,
  volumeSnapshot (`longhorn-velero`). Retention via
  `configs/cnpg-backup-retention.yaml`. This is the DB (orgs/users/sites/
  resources/policies/license registration).
- **Velero** `pangolin-bimonthly`: 15th & 30th 04:45 UTC → B2, ttl 28d (keep
  last 2). Catches the Gerbil WG-key PVC, the Traefik ACME-state PVC, and k8s
  objects. See [[backup-strategy]].

---

## Decisions & gotchas (chart is alpha — read before editing)

- **Image:** the chart auto-selects a `postgresql-<AppVersion>` image for
  Postgres mode (would be `postgresql-1.18.2`). We override with
  `images.pangolin.tag: ee-postgresql-1.19.2` (the one tag that is EE **and**
  Postgres-capable **and** 1.19). Tag families on Docker Hub:
  `1.x` (community/sqlite), `postgresql-1.x` (community/pg), `ee-1.x` (EE/sqlite),
  `ee-postgresql-1.x` (EE/pg ← this one).
- **The chart's Traefik does not work — we run our own.** Discovered on first
  deploy (2026-06-15):
  - *standalone* Traefik (`deployment.type=standalone`, `traefik.enabled=true`)
    hardcodes its args with **no admin/ping entrypoint**, so the `:8085/ping`
    probe gets connection-refused → permanent CrashLoop; it also hardcodes
    `--providers.kubernetescrd/kubernetesingress` (needs Traefik CRDs + a SA
    token + the kube-controller — none present) instead of Pangolin's HTTP
    provider.
  - *controller* mode ships **no Traefik subchart** in `0.1.0-alpha.0/.1`
    (`charts/` has only the CNPG deps), so `installTraefikController=true`
    deploys no Traefik at all.
  So `traefik.enabled=false`, `controller.enabled=false`, and the edge is a
  hand-authored Traefik (08–10) using the canonical Pangolin compose wiring:
  `providers.http` → `http://pangolin:3001/api/v1/traefik-config` + a file
  provider for the dashboard routers + the **Badger** plugin (`v1.4.1`, fetched
  at startup — needs egress to GitHub). No Kubernetes RBAC/CRDs needed. ACME is
  **DNS-01** (HTTP-01 can't work for an internal-only domain) using the
  `pangolin-traefik-cloudflare` token via `CF_DNS_API_TOKEN`. Pangolin still
  GENERATES resource routes because `pangolin.config.traefik.enabled=true`
  (entrypoints `web`/`websecure`, resolver `letsencrypt` must match our Traefik).
  - **Badger version** tracks Pangolin's installer, which builds with the
    *latest* badger tag (`make` does `jq '.[0].name'`). Bump `v1.4.1` in
    `08-traefik-config.yaml` when upgrading Pangolin.
  - ACME state persisted on the **shared RWX** `pangolin-acme` PVC (`07`,
    nfs-truenas): edge mounts it RW at `/letsencrypt`, the **server** pod mounts it
    RO at `/app/config/letsencrypt`. RWX is required because Pangolin's
    `acmeCertSync` (server side) scrapes Traefik's `acme.json` to flip a resource's
    cert status **pending → valid**; with the edge's old RWO PVC the server warned
    `cannot stat config/letsencrypt/acme.json` every 5s and every resource cert was
    stuck "pending" (resolved 2026-06-16). The server mount is injected via a Flux
    **postRenderer** kustomize patch (`06`), NOT `pangolin.extraVolumes` — the
    0.1.0-alpha.0 chart renders `extraVolumeMounts` with a broken
    `- {{- toYaml . | nindent 12 }}` that emits invalid YAML and fails the upgrade.
    Migrated the existing `acme.json` volume-to-volume (busybox pod mounting both
    PVCs) so no certs re-issued. Old `pangolin-edge-acme` + a stale
    `pangolin-traefik-acme` PVC are unreferenced orphans (kustomization `prune:
    false`) — delete manually. NOTE: a resource only goes "valid" once the **edge
    actually issues its cert**; the edge's HTTP provider must emit a router for the
    resource's host (lazy ACME on first request) — `acmeCertSync` only reflects
    what's already in `acme.json`.
  - **DNS-01 propagation check** (`08`, `dnsChallenge.propagation`) needs two
    non-default settings to issue from in-cluster pods (resolved 2026-06-16):
    `disableANSChecks: true` — lego's default check queries the zone's
    *authoritative* Cloudflare NS (e.g. `santino.ns.cloudflare.com:53`) directly,
    which is unreachable from pods here (timed out → "did not return the expected
    TXT"); and `delayBeforeChecks: "120s"` — with the authoritative check off lego
    stopped waiting and asked LE within ~13s, so LE got "No TXT record found"
    before the record propagated. The delay lets the Cloudflare TXT propagate to
    LE's resolvers. `resolvers: 1.1.1.1/1.0.0.1` (reachable from pods) back the
    recursive check. Verified: edge serves a real LE cert, `curl` ssl_verify=0.
  - Traefik runs as root to bind `:80/:443` (ns PSA is already `privileged`).
- **NetworkPolicies disabled** (`networkPolicy.enabled: false`). k3s enforces
  NetworkPolicy by default, and the chart's policies are written for the chart's
  *embedded* Postgres + controller topology: under enforcement they would block
  Pangolin → our external CNPG (pod-label mismatch on the 5432 egress rule) and
  have **no HTTPS egress** (breaking external OIDC to Authentik and the EE
  license check at app.pangolin.net). Rest of the cluster runs without per-app
  NetworkPolicies, so we match. **TODO:** author correct policies for this
  topology if we want default-deny here.
- **Dashboard IngressRoute disabled** (`pangolin.ingressRoute.dashboard.enabled:
  false`) — that's a controller-mode (Traefik CRD) feature. The dashboard is
  routed by our Traefik **file provider** (`dynamic_config.yml`: `next-service`
  → `pangolin:3002`, `api-service` → `pangolin:3000`, all behind `badger`). If
  the dashboard host changes, update the `Host(...)` rules in that ConfigMap.
- **Telemetry / update notifications disabled** to match the cluster's privacy
  posture (cf. Authentik error-reporting off).
- **EE vs community is a tag swap**, not a one-way door — data is identical, so
  we can drop to `postgresql-1.19.x` (community) by changing `images.pangolin.tag`
  if EE is ever dropped.

See also: [[cnpg-strategy]] (DB instance-count rationale), [[guacamole]]
(parallel RDP/VNC stack), [[backup-strategy]].
