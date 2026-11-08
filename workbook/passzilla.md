# Passzilla (Password Pusher)

Self-hosted [Password Pusher](https://github.com/pglombardo/PasswordPusher)
(`pwpush`): paste a password or note, get a link that expires after N views or N
days. Used to hand secrets to people outside the vault. Added 2026-05-13.

- **Namespace:** `passzilla`
- **Hostnames:**
  - LAN: https://passzilla.local.safeqbit.com (ingress-nginx)
  - Public: https://passzilla.safeqbit.com (Cloudflare tunnel `passzilla-cf-tunnel`; see [Exposure](#exposure))
- **Manifests:** `apps/safeqbit-local-hq/passzilla/`, tunnel in `apps/safeqbit-local-hq/cloudflared/tunnel-passzilla.yaml`
- **Image:** `pglombardo/pwpush:2.16.0` (Docker Hub, pinned by tag, not digest)
- **Storage:** `passzilla-db`, 1Gi Longhorn RWO at `/opt/PasswordPusher/storage/db`
- **Database:** SQLite, `production.sqlite3` on that volume. This includes the Solid Queue job tables
- **Backup:** `infrastructure/.../velero-schedule-passzilla.yaml`, 11th + 26th 03:00 UTC to B2, 28d retention

---

## What's in the namespace

| Object | Name | Notes |
|---|---|---|
| Deployment | `passzilla` | 1 replica, `Recreate`, 60s node-down tolerations, `fsGroup: 1000` (the image runs as uid 1000 `pwpusher`). Requests 50m / 512Mi, **limit 1Gi**. `/` readiness and liveness probes on :5100 |
| PVC | `passzilla-db` | `longhorn`, 1Gi, RWO |
| ConfigMap | `passzilla-config` | `PWP__*` settings, loaded with `envFrom` (below) |
| SealedSecret → Secret | `passzilla-secret` | One key: `SECRET_KEY_BASE` |
| Service | `passzilla` | ClusterIP :80 → 5100 |
| Ingress | `passzilla` | ingress-nginx, `letsencrypt-prod`, TLS secret `passzilla-tls`, host `passzilla.local.safeqbit.com` |

ConfigMap keys:

| Key | Value | Effect |
|---|---|---|
| `PWP__HOST_DOMAIN` | `passzilla.safeqbit.com` | The host used in generated links. It is the **public** name, so links work for people outside the LAN |
| `PWP__DISABLE_LOGINS` | `"true"` | No user accounts at all |
| `PWP__DISABLE_SIGNUPS` | `"true"` | |
| `PWP__ALLOW_ANONYMOUS` | `"true"` | Anyone who can reach it can create a push |

`SECRET_KEY_BASE` keeps Rails sessions and cookies valid across restarts.
Without it pwpush generates a random one at every boot and logs that it did.

The pod runs web (Puma) and worker (Solid Queue background jobs) under foreman,
and idles at ~480–560Mi.

## Exposure

Unlike most apps here, passzilla is meant to be reached from outside the LAN:
the recipient of a push usually isn't on it.

- **Tunnel:** `passzilla-cf-tunnel` in the `cloudflared` namespace, with 2
  connector pods, a PDB, a ServiceMonitor and the token sealed in
  `passzilla-cf-tunnel-token`. The public hostname and its origin
  (`http://passzilla.passzilla.svc.cluster.local`, per
  [cloudflared-migration.md](cloudflared-migration.md)) are set in the
  Cloudflare Zero Trust dashboard, not in Git. Traffic from the tunnel goes
  straight to the Service and skips ingress-nginx.
- **Cloudflare Access policy, if any, is also dashboard-only.** Nothing in
  this repo enforces or records it. A recipient has to be able to open a push
  link, so check what the policy allows from an incognito window after
  any dashboard change.
- **Client IPs.** Since 2.16.2/2.17.0 pwpush honours forwarded headers only
  from trusted proxy peers. `PWP__TRUSTED_PROXIES` is unset, and the
  defaults cover `10.0.0.0/8`, which includes the pod network (`10.42.0.0/16`,
  where both ingress-nginx and the cloudflared connectors live). If
  the audit log ever shows `10.42.x.x` addresses instead of real clients, that
  setting is the place to look.

## Payload encryption uses the default key

**`PWPUSH_MASTER_KEY` is not set.** pwpush encrypts push payloads at rest with
Lockbox, and with no key configured it falls back to a hard-coded default that
is published in its source (`config/initializers/lockbox.rb`). Anything sitting
in `production.sqlite3`, and so in every Velero backup on B2, is encrypted with
a key anyone can look up. Short expiries (1 day / 1 view) limit how much is in
there at any moment.

Fixing it is a config change with one trap: **changing the key makes existing
pushes unreadable** unless the old key is kept as a previous key. The safe
version:

1. Generate a key (`openssl rand -hex 32`, or the instance's own
   `/pages/generate_key` page).
2. Seal `PWPUSH_MASTER_KEY=<new>` into `passzilla-secret`, together with the
   existing `SECRET_KEY_BASE`, and add
   `PWPUSH_MASTER_KEY_PREVIOUS=<the default key from lockbox.rb>`. Both are
   read in 2.16.0.
3. After the longest push expiry has passed, drop `PWPUSH_MASTER_KEY_PREVIOUS`.

Not done yet. Noted here so it doesn't get forgotten.

## Single replica

SQLite on an RWO volume: one writer, `Recreate`, 60s tolerations, same as
vaultwarden and uptime-kuma. Don't scale it. A rollout means a few seconds of
downtime.

## What lives on the volume

```
production.sqlite3 (+ -wal, -shm)   pushes, audit log, Solid Queue tables
first_run_boot_code.txt             pwpush first-run code (treat as a secret)
db/production.sqlite3               stale, from 2026-05-13 (see below)
```

The mount path changed twice on the first day (`/opt/PasswordPusher/db` →
`/storage` → `/storage/db`). The `db/` subdirectory is left over from the middle
step and hasn't been written since 2026-05-13. The live database is the one at
the volume root.

```sh
kubectl -n passzilla exec deploy/passzilla -- ls -la /opt/PasswordPusher/storage/db
```

## Upgrading

1. Read the release notes (`gh release view v<version> -R pglombardo/PasswordPusher`).
   GitHub tags have a `v`, Docker Hub tags don't. Look for:
   - **migrations** (`db/migrate` in the diff). The container runs
     `db:migrate` at startup ("Password Pusher: migrating database to
     latest…" in the log) and nothing migrates back down, so a rollback across a
     migration means restoring the volume.
   - config renames under `PWP__*`
   - memory growth (see below)
2. Take a restore point if anything migrates:
   ```sh
   velero backup create passzilla-pre-<version>-$(date +%Y%m%d) --from-schedule passzilla-bimonthly
   velero backup get | grep passzilla-pre        # wait for Completed
   ```
3. Bump the tag in `03-deployment.yaml`, commit, open a PR, merge.
4. Check it:
   ```sh
   kubectl -n passzilla logs deploy/passzilla | grep -E 'Version|migrat'
   kubectl -n passzilla get pods     # RESTARTS stays 0
   ```
   Create a test push from the public hostname and open it.

**Watch memory after an upgrade.** 2.7.0 on a 512Mi limit was OOM-killed every
~15 minutes for 22 days (1,671 restarts) before anyone noticed. The limit has
been 1Gi since 2026-07-03 (PR #17). Steady state on 2.16.0 is ~560Mi. If a new
release pushes it towards 1Gi, raise the limit before it starts flapping:
`kubectl -n passzilla top pod`.

Version history in Git: `latest` (2026-05-13) → 2.7.0 (pinned 2026-05-28) →
2.9.3 (07-19) → 2.11.5 (08-29) → 2.14.0 (09-22) → 2.16.0 (10-23).

## Restore

Pushes are short-lived by design, so losing a few days of them is rarely a
problem. The audit log is the only thing worth having back.

| Source | Age |
|---|---|
| Longhorn `weekly-snapshot` (Sun 04:15, retain 1) | ≤ 1 week, in-cluster |
| Velero `passzilla-bimonthly` on B2 (28d TTL) | ≤ ~15 days, off-site |

Passzilla is the guinea pig for Velero restore drills, and
[restore-drills.md](restore-drills.md) Drill B has the full procedure in both
forms:

- **B1, non-destructive:** `--namespace-mappings passzilla:passzilla-drill`,
  then check through `kubectl -n passzilla-drill port-forward deploy/passzilla 8080:5100`.
- **B2, replace the live namespace:** `flux suspend kustomization apps`, delete
  the namespace, `velero restore create --from-backup <backup> --include-namespaces passzilla`,
  resume Flux.

After a full restore:

- **Adopt the orphaned `passzilla-secret`.** Velero brings the Secret back
  without its SealedSecret owner reference (Drill B gotchas). The app works
  regardless, because the restored Secret has the right `SECRET_KEY_BASE`.
- **Backups from before 2026-07-18** include cert-manager
  `CertificateRequest`s, which stall the next `passzilla-tls` renewal. Add
  `--exclude-resources certificaterequests.cert-manager.io,orders.acme.cert-manager.io,challenges.acme.cert-manager.io`.
- The tunnel lives in the `cloudflared` namespace and is not part of this
  restore. Flux recreates it from Git, and its token is a SealedSecret.
