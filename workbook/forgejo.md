# Forgejo — offline GitHub mirror

An offline, LAN-side copy of every GitHub repo I own: the `fqazzazee` account
and the `Zonduit` org. Each repo is a read-only pull mirror; Velero ships the
whole thing to B2 with two retention tiers. Added 2026-11-15.

- **Namespace:** `forgejo`
- **Hostname:** https://git.local.safeqbit.com (LAN only, login required for everything)
- **Manifests:** `apps/safeqbit-local-hq/forgejo/`
- **Image:** `codeberg.org/forgejo/forgejo:15.0.11-rootless`, pinned by digest. 15.x is the LTS line
- **Storage:** `forgejo-data` 5Gi Longhorn RWO at `/var/lib/gitea`: repos, SQLite DB and `app.ini` (with its generated secrets)
- **Backup:** `infrastructure/.../velero-schedule-forgejo.yaml`, `forgejo-daily` (04:30, 14 kept) and `forgejo-monthly` (3rd 04:45, 12 kept), both to B2
- **Login:** Authentik OIDC, plus a local break-glass admin `forgejo-admin` (password in Vaultwarden)

---

## What this protects against, and what it doesn't

```
GitHub ──(pull mirror, hourly)──► Forgejo ──(Velero: 14 daily + 12 monthly)──► B2
```

| Scenario | Covered by |
|---|---|
| GitHub down, or no internet | Forgejo: clone over HTTPS from the LAN |
| Repo deleted on GitHub / account locked out | Forgejo: sync fails, mirror keeps its last state |
| **Force-push or branch delete on GitHub** | **Velero only.** The mirror copies it within the hour (`git remote update --prune`). Restore from a daily (≤14 days ago) or monthly (≤12 months ago) |
| Cluster lost | Velero restore from B2 (GitHub itself is still the other copy) |

**The mirror is not the backup. The B2 copies are.** Keep that in mind before
changing the Velero TTLs.

**Out of scope:** issues, PRs, releases and other GitHub metadata. A pull
mirror only carries git data (all refs, the wiki, LFS objects).

---

## Design notes

**One volume, SQLite, no CNPG.** One user, few writes, and the mirror
definitions can be recreated by re-running Gickup, so a Postgres cluster would
only add moving parts. Keeping the DB, the repos and `app.ini` in one volume
means each Velero backup is a consistent set.

**Secrets are not in Git.** `SECRET_KEY`, `INTERNAL_TOKEN` and `JWT_SECRET`
are generated on first boot into `/var/lib/gitea/custom/conf/app.ini` on the
PVC. `SECRET_KEY` encrypts DB columns (the OIDC client secret, mirror
credentials), so it has to be backed up with the DB, and here it always is.
**Never delete `app.ini` by hand:** a new key can't decrypt the existing rows.

**Config is env-only.** Every setting is a `FORGEJO__section__KEY` env var in
`03-deployment.yaml`. On every start the image's `environment-to-ini` writes
them into `app.ini`, so Git wins over anything edited in the file. To change a
setting, change the env var, not the file.

**Lockdown:**
- `REQUIRE_SIGNIN_VIEW=true`: anonymous web → login page, anonymous API → 403
- local sign-up refused (`ALLOW_ONLY_EXTERNAL_REGISTRATION`); new accounts only via Authentik, and the Authentik application is bound to my user
- `FORCE_PRIVATE=true`: every repo is private
- `migrations.ALLOWED_DOMAINS=github.com`: mirrors can only pull from GitHub (this also stops the migrate API being used as an SSRF tool against the LAN)
- no SSH server, no Actions, no package registry, no update checker, `OFFLINE_MODE` (no Gravatar)

All of this was checked against this exact image digest in a local podman run
(read-only rootfs, uid 1000) before the first deploy.

**`dnsConfig ndots:1`** because the image is Alpine (musl). Without it,
`github.com` is looked up through the `local.safeqbit.com` search domain
first and can hard-fail (see the DNS search-amplification note in maintenance.md).

---

## First-time setup (after the PR merges)

### 1. DNS

On the router: `git.local.safeqbit.com` CNAME → `ingress.local.safeqbit.com`.

```bash
kubectl -n forgejo get pods,pvc,ingress,certificate
curl -sI https://git.local.safeqbit.com/ | head -1      # 303 → /user/login
```

### 2. Break-glass admin

```bash
kubectl -n forgejo exec deploy/forgejo -- forgejo admin user create \
  --admin --username forgejo-admin --email forgejo-admin@localhost \
  --random-password --random-password-length 32 --must-change-password=false
```

Save the printed password in Vaultwarden (`Forgejo break-glass`). This
account owns the Gickup API token too (see Mirroring below). Use it for
nothing else.

### 3. Authentik provider + application

In `authentik01.local.safeqbit.com` → Admin:

1. **Applications → Providers → Create → OAuth2/OpenID Provider**
   - Name: `forgejo`
   - Authorization flow: `default-provider-authorization-implicit-consent`
   - Client type: Confidential
   - Redirect URI (strict): `https://git.local.safeqbit.com/user/oauth2/authentik/callback`
   - **Signing key: `authentik Self-signed Certificate`.** Without one, tokens are HS256 and Forgejo rejects them
   - Note the Client ID and Client Secret
2. **Applications → Create**: name `Forgejo`, slug `forgejo`, provider `forgejo`,
   launch URL `https://git.local.safeqbit.com/`
3. **Policy / Group / User bindings** on that application: bind my user only.
   This is what keeps anyone else in Authentik from creating a Forgejo account
4. **Directory → Groups → Create** `Forgejo Admins`, add my user. Authentik's
   `profile` scope already sends a `groups` claim

### 4. Forgejo auth source

Log in as `forgejo-admin` → Site administration → Identity & access →
Authentication sources → Add:

| Field | Value |
|---|---|
| Authentication type | OAuth2 |
| Authentication name | `authentik` (must match the `/user/oauth2/authentik/callback` path) |
| OAuth2 provider | OpenID Connect |
| Client ID / Secret | from step 3 |
| Auto discovery URL | `https://authentik01.local.safeqbit.com/application/o/forgejo/.well-known/openid-configuration` |
| Additional scopes | `email profile` |
| Claim name providing group names | `groups` |
| Group claim value for administrator users | `Forgejo Admins` |

Saving runs OIDC discovery from the pod, so a save error means the pod can't
reach Authentik (it hairpins through 10.10.13.50, like AFFiNE).

### 5. First Authentik login

Log out → **Sign in with authentik**. With auto-registration off, Forgejo
shows a "register new account" form: pick username **`fqazzazee`**, so
`git.local.safeqbit.com/fqazzazee/<repo>` matches
`github.com/fqazzazee/<repo>`. The `Forgejo Admins` group makes it an admin
on every login. The Zonduit repos go under a Forgejo org named `Zonduit`.

Check that the break-glass login still works afterwards (password form on
`/user/login`).

---

## Mirroring

The Gickup CronJob that creates and maintains the mirrors comes in a
follow-up PR. Until then the instance is empty. The plan:

- **GitHub credentials:** two fine-grained PATs, read-only **Contents** +
  **Metadata** on all repos. One is for `fqazzazee`, the other for `Zonduit`:
  a fine-grained PAT can only target one owner, and the org may need to
  approve it. Both go in SealedSecrets.
- **Forgejo credentials:** an API token of `forgejo-admin`, also sealed.
- Gickup runs daily, lists both owners and creates any missing pull mirror
  (hourly interval). Forgejo does the actual syncing.
- **PAT expiry is the most likely way this breaks without anyone noticing.**
  Mirrors keep the credentials they were created with, so check during the
  build whether rotation needs a per-mirror update. A stale-mirror check
  goes into `backup-summary-report`.

---

## Verify

```bash
# pod healthy, volume usage
kubectl -n forgejo get pods
kubectl -n forgejo exec deploy/forgejo -- du -sh /var/lib/gitea/git/repositories

# both schedules exist; trigger a first backup instead of waiting for 04:30
velero schedule get | grep forgejo
velero backup create --from-schedule forgejo-daily
velero backup describe <name> --details | grep -E "Phase|forgejo-data"
```

The Monday `backup-summary-report` picks up both schedules automatically.

---

## Restore

**One repo to an earlier state** (e.g. after an upstream force-push):
restore into a scratch namespace, clone from there, push wherever needed.

```bash
velero restore create forgejo-scratch --from-backup <backup> \
  --namespace-mappings forgejo:forgejo-restore
kubectl -n forgejo-restore port-forward deploy/forgejo 3001:3000
# the restored Ingress points at the real hostname: delete it
kubectl -n forgejo-restore delete ingress forgejo
git clone http://localhost:3001/fqazzazee/<repo>.git   # log in as forgejo-admin
kubectl delete ns forgejo-restore
```

The restored copy still has its mirror settings and will start pulling from
GitHub on the hourly schedule, so **clone what you need before the next sync
runs** (or turn off the restored mirrors' sync in their repo settings first).

**Whole instance:** scale the Deployment to 0, delete the PVC, `velero restore
create --from-backup <backup> --include-namespaces forgejo`, and let Flux
reconcile.

---

## Upgrades

Stay on the LTS line (15.x) and bump patch releases by changing the tag **and**
the digest:

```bash
skopeo inspect --format '{{.Digest}}' docker://codeberg.org/forgejo/forgejo:15.0.<n>-rootless
```

Before a major (e.g. to the next LTS), take a restore point
(`velero backup create --from-schedule forgejo-daily`). Forgejo migrates the
DB on boot and downgrades are not supported. Read the release notes for
`app.ini` keys that changed name.
