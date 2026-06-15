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

Topology: **`deployment.type=standalone`, `mode=multi`** — mirrors Pangolin's
proven docker-compose stack (three separate Deployments) rather than the
controller + Traefik-CRD mode. Lowest risk on the alpha Helm chart.

| Component | Image | Exposure / notes |
|---|---|---|
| **Pangolin** (app/API/dashboard) | `fosrl/pangolin:ee-postgresql-1.19.2` | ClusterIP; behind Traefik. EE + Postgres-capable + 1.19. |
| **Traefik** (Pangolin's own edge) | `traefik:v3.6.15` | **LoadBalancer `10.10.13.51`**; does its **own ACME** (Cloudflare DNS-01), **per-host** certs (`prefer_wildcard_cert: false`). Does NOT use nginx-ingress/cert-manager. |
| **Gerbil** (WireGuard) | `fosrl/gerbil:1.3.1` | **LoadBalancer `10.10.13.52`**, UDP 51820/21820; `NET_ADMIN`; PVC for WG key. |
| **Database** | CNPG `pangolin-cnpg` | Hand-authored Cluster (Longhorn 5Gi, `instances:1`), CNPG app secret `pangolin-cnpg-app` key `uri`. |

Deployed via Flux **HelmRelease** against the official OCI chart
`oci://ghcr.io/fosrl/helm-charts/pangolin:0.1.0-alpha.0` (source `fossorial` in
`infrastructure/.../controllers/sources.yaml`). The chart's AppVersion is 1.18.2;
the EE/1.19 image is pinned via `images.pangolin.tag`.

### MetalLB IPs

`10.10.13.50` = ingress-nginx (existing). Pangolin pins `10.10.13.51` (Traefik)
and `10.10.13.52` (Gerbil) from the **reserved** pool (`10.10.13.50–59`,
`autoAssign:false`) via the `metallb.io/loadBalancerIPs` annotation.

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
   - Gerbil uses the raw IP `10.10.13.52`, no DNS record needed.
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
- **Traefik ACME PVC is NOT chart-managed in standalone mode.** The chart's
  standalone Traefik mounts `pangolin-traefik-acme` but ships no template to
  create it (only Gerbil's PVC is templated). We create it ourselves
  (`07-traefik-acme-pvc.yaml`) and set `traefik.persistence.existingClaim`.
  Without it Traefik would hang Pending; without persistence at all it would
  re-issue certs each restart and risk hitting LE's rate limits.
- **`traefik.config.dynamicRouters.host` is required** (chart validation
  `PANGOLIN-043`) — set to the dashboard host.
- **NetworkPolicies disabled** (`networkPolicy.enabled: false`). k3s enforces
  NetworkPolicy by default, and the chart's policies are written for the chart's
  *embedded* Postgres + controller topology: under enforcement they would block
  Pangolin → our external CNPG (pod-label mismatch on the 5432 egress rule) and
  have **no HTTPS egress** (breaking external OIDC to Authentik and the EE
  license check at app.pangolin.net). Rest of the cluster runs without per-app
  NetworkPolicies, so we match. **TODO:** author correct policies for this
  topology if we want default-deny here.
- **Dashboard IngressRoute disabled** (`pangolin.ingressRoute.dashboard.enabled:
  false`) — that's a controller-mode (Traefik CRD) feature; in standalone mode
  Pangolin's own dynamic Traefik config routes the dashboard and the IngressRoute
  CRD isn't installed.
- **Telemetry / update notifications disabled** to match the cluster's privacy
  posture (cf. Authentik error-reporting off).
- **EE vs community is a tag swap**, not a one-way door — data is identical, so
  we can drop to `postgresql-1.19.x` (community) by changing `images.pangolin.tag`
  if EE is ever dropped.

See also: [[cnpg-strategy]] (DB instance-count rationale), [[guacamole]]
(parallel RDP/VNC stack), [[backup-strategy]].
