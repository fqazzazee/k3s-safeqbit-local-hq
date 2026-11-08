# Vaultwarden

Bitwarden-compatible password manager. The official Bitwarden apps and browser
extensions talk to it as a self-hosted server. Added 2026-05-06 on NFS and moved
to Longhorn the next day.

It is also where the keys to everything else are kept, including the
cluster's own disaster-recovery keys. Read
[The chicken-and-egg problem](#the-chicken-and-egg-problem) before you need it.

- **Namespace:** `vaultwarden`
- **Hostname:** https://vaultwarden.local.safeqbit.com (LAN only. No cloudflared tunnel and no public record)
- **Manifests:** `apps/safeqbit-local-hq/vaultwarden/`
- **Upstream:** https://github.com/dani-garcia/vaultwarden
- **Image:** `vaultwarden/server:1.37.4` (Docker Hub, pinned by tag, not digest). Bundles its own copy of the web vault
- **Storage:** `vaultwarden-data`, 5Gi Longhorn RWO at `/data` (~6MB used on 2026-11-08)
- **Database:** SQLite, `/data/db.sqlite3` (WAL mode). No CNPG
- **Backup:** `infrastructure/.../velero-schedule-vaultwarden.yaml`, 7th + 22nd 03:00 UTC to B2, **180d** retention (the longest in the cluster, on purpose)

---

## What's in the namespace

| Object | Name | Notes |
|---|---|---|
| Deployment | `vaultwarden` | 1 replica, `Recreate`, 60s node-down tolerations. Requests 50m / 128Mi, limit 512Mi. `/alive` readiness and liveness probes. Runs as root (the image default) |
| PVC | `vaultwarden-data` | `longhorn`, 5Gi, RWO. The comments in `01-namespace-pvc.yaml` still describe the old `nfs-truenas` setup. `storageClassName: longhorn` is what counts |
| ConfigMap | `vaultwarden-config` | Non-secret settings, loaded with `envFrom` (below) |
| SealedSecret → Secret | `vaultwarden-secrets` | One key: `ADMIN_TOKEN` |
| Service | `vaultwarden` | ClusterIP :80 |
| Ingress | `vaultwarden` | ingress-nginx, `letsencrypt-prod`, TLS secret `vaultwarden-tls`. `proxy-read-timeout`/`proxy-send-timeout` 3600 for the WebSocket notifications, `proxy-body-size` 100m for imports and attachments |

ConfigMap keys:

| Key | Value | Why |
|---|---|---|
| `DOMAIN` | `https://vaultwarden.local.safeqbit.com` | Must match the Ingress host. Used in links and for WebAuthn |
| `SIGNUPS_ALLOWED` | `"false"` | Closed. See [Adding a user](#adding-a-user) |
| `INVITATIONS_ALLOWED` | `"false"` | There is no SMTP configured, so invitations couldn't be delivered anyway |
| `LOG_LEVEL` | `warn` | |
| `IP_HEADER` | `X-Forwarded-For` | Real client IPs in logs and rate limiting, behind ingress-nginx |

Login is Vaultwarden's own accounts and master passwords. There is no
Authentik in front. `/admin` is enabled because `ADMIN_TOKEN` is set, and the
token is the only thing protecting it.

## What lives in `/data`

```
db.sqlite3 (+ -wal, -shm)   the vault: users, ciphers (end-to-end encrypted), orgs
rsa_key.pem                 signs login tokens. Lose it and every client is logged out
attachments/  sends/        file attachments and Send files
icon_cache/                 website favicons, rebuildable
config.json                 settings saved from the admin panel (see below)
```

**`config.json` overrides the environment.** Anything saved from the admin
panel lands here and wins over the ConfigMap. The startup log says so:
`Using saved config from data/config.json for configuration`. If a ConfigMap
change seems to do nothing, check whether that key is in `config.json`. On
2026-11-08 the file held only Vaultwarden's own generated `_duo_akey` and no
admin-panel settings. To list the keys without printing values:

```sh
kubectl -n vaultwarden exec deploy/vaultwarden -- grep -o -E '"[a-z_]+"[[:space:]]*:' /data/config.json
```

The NFS-era directory (`/mnt/nvme2tb/k8s/pvs/vaultwarden-vaultwarden-data` on
TrueNAS, with a May `db.sqlite3`) was still there in September. If it still
is, it's a stale leftover and not a backup of the current vault. Don't restore
from it.

## Single replica, permanently

SQLite on an RWO volume means a single writer. `Recreate`, so the old pod
releases the volume before the new one starts. A rollout takes the vault offline
for a few seconds, and the apps carry on from their local cache meanwhile.
Resilience is reschedule speed (60s tolerations plus Longhorn's
`nodeDownPodDeletionPolicy`), the same as uptime-kuma, passzilla and affine.
Running more than one replica would mean moving to Postgres first (notes in
[cnpg-strategy.md](cnpg-strategy.md) → History). That isn't planned.

## Adding a user

Signups and invitations are both off, and there is no SMTP. The way this has
been done (commits `43421ff` → `a8da9f3`):

1. Set `SIGNUPS_ALLOWED: "true"` in `03-configmap.yaml`, commit, push.
2. **Restart the pod.** `envFrom` is read only at container start, and
   nothing rolls the Deployment when the ConfigMap changes:
   `kubectl -n vaultwarden rollout restart deploy/vaultwarden`.
3. Create the account from the web vault.
4. Set it back to `"false"`, commit, push, restart again.

Don't flip it from the admin panel. That writes `config.json`, which then
overrides the ConfigMap for good.

## Upgrading

1. Read the release notes (`gh release view <tag> -R dani-garcia/vaultwarden`),
   looking for config changes and the bundled web-vault version. Vaultwarden
   runs its DB migrations itself at startup, and there is no down-migration, so
   **rolling back across a migration means restoring `/data`**, not just
   reverting the tag.
2. Take an off-site restore point:
   ```sh
   velero backup create vaultwarden-pre-<version>-$(date +%Y%m%d) --from-schedule vaultwarden-bimonthly
   velero backup get | grep vaultwarden-pre       # wait for Completed
   ```
   The local `velero` CLI works. No need to exec into the Velero pod, and
   `deploy/velero` can land on a node-agent pod anyway. If B2 is unavailable
   (it has happened: daily cap), a Longhorn VolumeSnapshot of
   `vaultwarden-data` will do for a short soak. That is what
   `vaultwarden-preupgrade-20260718` was.
3. Change the `image:` tag in `04-deployment.yaml`, commit, open a PR, merge.
4. Check it came up on the new version and isn't crash-looping:
   ```sh
   kubectl -n vaultwarden get pods
   kubectl -n vaultwarden logs deploy/vaultwarden | head -20   # "Version x.y.z", no config error
   ```
   Then log in from the web vault and one app.

**Since 1.37.5, invalid config values stop startup.** Values that used to be
clamped or ignored are now a fatal error: numbers out of range, a rate-limit
`*_SECONDS`/`*_MAX_BURST` or `DATABASE_TIMEOUT` of 0, `DATABASE_MIN_CONNS` above
`DATABASE_MAX_CONNS`, or a malformed cron in `DUO_CONTEXT_PURGE_SCHEDULE` /
`PURGE_INCOMPLETE_SSO_AUTH`. That applies to the ConfigMap **and** to
`config.json`. Our five ConfigMap keys contain none of those, and `config.json`
currently holds nothing but `_duo_akey`. Re-check both before any upgrade if
either has grown.

What a bad value looks like with `Recreate`: the old pod is already gone, the
new one exits on startup, and the vault is down until it's fixed. Apps keep
working read-only from their cache. The fastest fix is to put the previous tag
back (it still accepts the old config). Then correct the value and upgrade again.
If the bad value is in `config.json`, `kubectl exec` won't help because the
container won't stay up. Scale to 0 (with `flux suspend kustomization apps`
first), mount the PVC in a throwaway pod, edit the file, then scale back up and
resume Flux.

Version history in Git: 1.32.0 (2026-05-06) → 1.36.0 (2026-07-18) → 1.37.2
(2026-08-29) → 1.37.3 (2026-09-22) → 1.37.4 (2026-10-23, High GHSA).

## The chicken-and-egg problem

Vaultwarden holds credentials the cluster's disaster recovery depends on:

- the **offline copy of the sealed-secrets private keys** (node-bootstrap.md
  Path B: "restore the sealed-secrets master key from the offline backup
  (Vaultwarden)")
- the **etcd-snapshot B2 key** (`k3s-safeqbit-etcd`, node-bootstrap.md)
- logins for the other apps (Pulse, Uptime Kuma, …)

And Vaultwarden runs **inside the cluster it is meant to help rebuild**. In a
full loss:

- Its own data is on B2 via Velero. Velero's B2 credentials in a rebuilt
  cluster come from a SealedSecret (`velero-cloud-credentials`), which needs
  the sealed-secrets key, which is in Vaultwarden.
- The etcd-restore path (Path A) brings the sealing key back, but it needs the
  etcd B2 key, which is in Vaultwarden. Logging in to the B2 console to mint a
  new key needs the B2 account login, wherever that is kept.

So "it's in Vaultwarden" only works if there is a way to read Vaultwarden with
the cluster down. What breaks the loop:

- **Bitwarden apps keep an encrypted offline copy of the vault** and unlock it
  with the master password, without the server. A logged-in desktop or phone
  app is the first thing to reach for. Don't log out of all of them at once,
  and don't use "Deauthorize sessions" casually.
- **An encrypted vault export kept outside the cluster** (not on Longhorn, not
  only in this repo's B2 bucket). Refresh it after anything DR-relevant
  changes.
- **The sealed-secrets keys rotate every 30 days.** On 2026-11-08 there were 7
  active keys, from `sealed-secrets-keyfxn6x` (2026-05-05) to
  `sealed-secrets-keynnwhs` (2026-11-01). A SealedSecret made since the last
  export needs a key the offline copy doesn't have. Re-export the whole set
  after each rotation:
  ```sh
  kubectl -n kube-system get secrets -l sealedsecrets.bitnami.com/sealed-secrets-key \
    -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp
  ```

Restore drill D (tabletop) in [restore-drills.md](restore-drills.md) walks
through this access chain. Do it with the assumption that Vaultwarden is
unreachable.

## Restore

There is no Layer 2 (no CNPG). Two sources:

| Source | Age | Survives |
|---|---|---|
| Longhorn `weekly-snapshot` (Sun 04:15, retain 1) | ≤ 1 week | Bad data, a bad upgrade. **Not** losing the volume |
| Velero `vaultwarden-bimonthly` on B2 (180d) | ≤ ~15 days | Everything, including losing the cluster |

The Velero copy is a crash-consistent CSI snapshot of a running SQLite
database in WAL mode. SQLite recovers that cleanly on open.

**Roll back to the weekly Longhorn snapshot** (fastest, in-cluster):
`flux suspend kustomization apps`, scale `vaultwarden` to 0, then revert the
volume to its `weekly-s-…` snapshot as in [backup-strategy.md](backup-strategy.md)
→ "Roll back a single PVC to a Longhorn snapshot". Scale back to 1 and resume
Flux. Everything written since the snapshot is lost.

**Look at a backup without touching the live vault** (the Drill B1 pattern in
[restore-drills.md](restore-drills.md)):

```sh
velero backup get | grep vaultwarden
velero restore create vw-inspect-$(date +%Y%m%d) \
  --from-backup vaultwarden-bimonthly-<timestamp> \
  --namespace-mappings vaultwarden:vaultwarden-inspect
kubectl -n vaultwarden-inspect port-forward deploy/vaultwarden 8080:80
```

The copy's Ingress claims the same host as the live one, so go through the
port-forward. `DOMAIN` still names the real hostname, so expect anything
origin-bound (WebAuthn, generated links) to misbehave there. Delete the
namespace afterwards: it is a full copy of the vault.

**Full restore** (live namespace lost or unusable). This is Drill B2:

```sh
flux suspend kustomization apps                 # or Flux recreates an empty PVC mid-restore
kubectl delete ns vaultwarden                   # wait for it to be gone
velero restore create vw-restore-$(date +%Y%m%d) \
  --from-backup vaultwarden-bimonthly-<timestamp> --include-namespaces vaultwarden
velero restore describe vw-restore-<yyyymmdd> --details     # DataDownload Completed
flux resume kustomization apps
```

Then:

- **The restored `vaultwarden-secrets` is orphaned from its SealedSecret.**
  Velero restores the Secret without its owner reference, so the controller
  won't update it. Adopt it as in restore-drills.md Drill B.
- **Backups from before 2026-07-18 carry cert-manager `CertificateRequest`s.**
  A restored one silently blocks the next renewal of `vaultwarden-tls` while
  the Certificate still reads Ready. That happened here in July. For those
  backups add
  `--exclude-resources certificaterequests.cert-manager.io,orders.acme.cert-manager.io,challenges.acme.cert-manager.io`.
  The schedule excludes them from 2026-07-18 on.
- `rsa_key.pem` comes back with the data, so clients stay logged in. They
  resync and drop anything newer than the backup.
