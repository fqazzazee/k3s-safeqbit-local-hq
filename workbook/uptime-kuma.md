# Uptime Kuma

Status / uptime monitoring for the `safeqbit-local-hq` cluster.
Added 2026-05-29.

- **Namespace:** `uptime-kuma`
- **Hostname:** https://uptime.local.safeqbit.com (admin UI *and* status pages, internal only)
- **Manifests:** `apps/safeqbit-local-hq/uptime-kuma/`
- **Image:** `louislam/uptime-kuma:1.23.16` (pinned; bump deliberately)
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

## Restore (DR)

Everything that defines the monitoring stack is in the `uptime-kuma-data` PVC.
Restore the namespace from the latest Velero backup:

```sh
velero backup get | grep uptime-kuma-weekly
velero restore create --from-backup uptime-kuma-weekly-<TIMESTAMP>
```

Then reconcile Flux so the Deployment/Service/Ingress match Git again.
