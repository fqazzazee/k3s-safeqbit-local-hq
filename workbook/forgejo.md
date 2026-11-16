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
- **Mirroring:** Gickup CronJob `gickup`, daily 04:00 UTC, creates the pull mirrors; Forgejo syncs them hourly
- **Drift check:** CronJob `mirror-check`, daily 03:15 UTC, compares every branch and tag with GitHub; failure → `KubeJobFailed` → Slack
- **⏰ GitHub PATs expire 2027-11-16 (fqazzazee) / 2027-11-17 (Zonduit).** Rotate during **2027-11-01 → 2027-11-09**. See [Token expiry](#token-expiry)

---

## What this protects against, and what it doesn't

```
GitHub ──(pull mirror, hourly)──► Forgejo ──(Velero: 14 daily + 12 monthly)──► B2
```

| Scenario | Covered by |
|---|---|
| GitHub down, or no internet | Forgejo: clone over HTTPS from the LAN |
| Repo deleted on GitHub / account locked out | Forgejo: sync fails, mirror keeps its last state. `mirror-check` flags it as `GONE` the next morning |
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
4. **Admin rights come from the existing `authentik Admins` group**, so no new
   group is needed. Authentik's `profile` scope already sends a `groups` claim.
   This means anyone who is an Authentik superuser is a Forgejo admin too,
   provided the step-3 binding lets them log in at all

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
| Group claim value for administrator users | `authentik Admins` |

Saving runs OIDC discovery from the pod, so a save error means the pod can't
reach Authentik (it hairpins through 10.10.13.50, like AFFiNE).

### 5. First Authentik login

Log out → **Sign in with authentik**. With auto-registration off, Forgejo
shows a "register new account" form: pick username **`fqazzazee`**, so
`git.local.safeqbit.com/fqazzazee/<repo>` matches
`github.com/fqazzazee/<repo>`. Membership of `authentik Admins` makes it an
admin; Forgejo re-evaluates that **at each login**, so after changing the
group setting, log out and back in. Admin matters here: the `Zonduit` org
is created and owned by `forgejo-admin`, so a non-admin `fqazzazee` can't
see its repos. The Zonduit repos go under a Forgejo org named `Zonduit`.

Check that the break-glass login still works afterwards (password form on
`/user/login`).

### 6. After the first Gickup run: join the Zonduit org

Gickup creates the `Zonduit` org as `forgejo-admin`, so `forgejo-admin` is its
only member. Site admins can open it, but it won't appear on `fqazzazee`'s
profile or dashboard until `fqazzazee` is a member. Add it to the Owners team
(org → Teams → Owners → Add member, or via the API):

```bash
T=$(kubectl -n forgejo get secret gickup-tokens -o jsonpath='{.data.forgejo}' | base64 -d)
TID=$(curl -s -H "Authorization: token $T" https://git.local.safeqbit.com/api/v1/orgs/Zonduit/teams \
  | python3 -c "import json,sys; print([t['id'] for t in json.load(sys.stdin) if t['name']=='Owners'][0])")
curl -s -o /dev/null -w '%{http_code}\n' -X PUT -H "Authorization: token $T" \
  https://git.local.safeqbit.com/api/v1/teams/$TID/members/fqazzazee   # 204
unset T
```

Done on 2026-11-16. A restore from Velero keeps the membership, but a
rebuilt-from-scratch instance needs this step again.

---

## Mirroring

A daily **Gickup** CronJob (`05-gickup-config.yaml`, `07-gickup-cronjob.yaml`,
04:00 UTC) lists both GitHub owners and creates a Forgejo **pull mirror**
(hourly) for any repo that doesn't have one yet. For existing mirrors it
just triggers a sync. Forgejo does the hourly fetching itself.

| GitHub | Forgejo | Token |
|---|---|---|
| `github.com/fqazzazee/*` (32 repos on 2026-11-15) | user `fqazzazee` | `github-fqazzazee` |
| `github.com/Zonduit/*` (7, all private) | org `Zonduit` (created on first run) | `github-zonduit` |

The three tokens live in the `gickup-tokens` SealedSecret
(`06-gickup-tokens.sealed.yaml`): the two GitHub PATs and `forgejo`, an API
token of `forgejo-admin`.

**Gotchas, all checked against v0.10.47 + Forgejo 15 on 2026-11-15:**

- **`mirror.enabled: false` is correct.** In Gickup's gitea destination,
  `true` means "clone and push a plain copy", and only the default creates a
  real pull mirror.
- **`mirrorinterval` must be written `1h0m0s`**, exactly as Forgejo returns
  it. Otherwise every run re-PATCHes the interval and counts it as a failure.
- **Register `fqazzazee` (step 5) before the first run.** Pair 1 targets that
  *user* with `createorg: false`, so an early run fails all 32 repos instead
  of creating an org that takes the username. Tested: 32 errors, no org
  created, exit 1.
- **Wikis** (NetScan-WoL and howdy-fedora-helper-script have one) are mirrored
  into the same Forgejo repo. The log line `starting backup for ….wiki.git`
  with nothing after it is Gickup skipping its separate wiki entry. That's
  expected.
- **Exit code:** any error → exit 1 → Job failed → stock `KubeJobFailed`
  alert. An expired or revoked PAT breaks the listing, so it shows up here
  within a day. `backoffLimit: 0` keeps it to one failed Job per day.

### Creating the tokens

GitHub → Settings → Developer settings → Fine-grained tokens, **two** tokens
(a fine-grained PAT can only target one resource owner):

| Name | Resource owner | Repository access | Permissions |
|---|---|---|---|
| `forgejo-mirror-fqazzazee` | fqazzazee | All repositories | Contents: Read-only (Metadata: Read-only is added automatically) |
| `forgejo-mirror-zonduit` | Zonduit | All repositories | same |

The Zonduit one may need approving under the org's Settings → Personal
access tokens → Pending requests.

**Expiration:** pick *No expiration* if GitHub offers it. These tokens are
read-only and scoped to one owner, they sit on a LAN-only instance, and
Forgejo stores the mirror copy encrypted (below). Rotating them means
re-creating every mirror (next section), so expiry has a real cost. If the
org enforces a lifetime, use the maximum and put a reminder a week before it.

The Forgejo token (after step 2 above):

```bash
kubectl -n forgejo exec deploy/forgejo -- forgejo admin user generate-access-token \
  --username forgejo-admin --token-name gickup --raw \
  --scopes write:repository,write:organization,read:user,write:admin
```

`write:admin` is in the tested scope set because the token creates repos
under another account (`fqazzazee`).

### Token expiry

| Token | Where | Expires | Rotate during |
|---|---|---|---|
| `forgejo-mirror-fqazzazee` (GitHub PAT) | `gickup-tokens` → `github-fqazzazee` | **2027-11-16 05:00 UTC** | 2027-11-01 → 2027-11-09 |
| `forgejo-mirror-zonduit` (GitHub PAT) | `gickup-tokens` → `github-zonduit` | **2027-11-17 05:01 UTC** | same session |
| `gickup` (Forgejo API token of `forgejo-admin`) | `gickup-tokens` → `forgejo` | never (Forgejo tokens have no expiry) | only if leaked |

Both PATs were issued on 2026-11-16 with a one-year lifetime, so unless a
rotation picks *No expiration*, this comes round every November. **Put a
calendar reminder on 2027-11-01.** The rotation window
ends a week before expiry so a failed attempt still leaves time.

What happens if it's missed: the next 03:15 `mirror-check` fails with
`ERROR … HTTP 401` and the 04:00 `gickup` run fails too, so the stock
`KubeJobFailed` alert reaches Slack within a day. The mirrors keep their
last state (nothing is lost) but stop updating until the rotation is done.

**Check the live expiry dates** (reads the tokens from the cluster secret,
prints only the date that GitHub returns in a response header):

```bash
for k in github-fqazzazee github-zonduit; do
  echo "$k: $(GH_TOKEN=$(kubectl -n forgejo get secret gickup-tokens -o jsonpath="{.data.$k}" | base64 -d) \
    gh api -i /user 2>/dev/null | grep -i '^github-authentication-token-expiration' | cut -d' ' -f2-)"
done
```

**Rotation checklist:**

1. GitHub → Fine-grained tokens → **Regenerate** each token (same
   permissions; see [Creating the tokens](#creating-the-tokens)). The
   Zonduit one may need re-approving in the org.
2. Re-seal `06-gickup-tokens.sealed.yaml` with both new PATs plus the
   unchanged `forgejo` token, via a PR. (The `forgejo` value can be read back
   from the live secret, so it doesn't need regenerating.)
3. Re-create the mirrors so they pick up the new token. Follow the steps
   below, because each mirror stores its own copy of the PAT.
4. Run `mirror-check` by hand and expect `39 in sync` (or the current count).
5. Update the dates in this table and in the summary at the top, and add a
   change-log row.

### Where the mirror credentials live, and rotating them

Forgejo stores each mirror's GitHub token **in the DB**
(`mirror.encrypted_remote_address`, encrypted with `SECRET_KEY`). The bare
repo's git config only has `https://oauth2@github.com/…`. So:

- the Velero copies hold the PATs only in encrypted form, and only together
  with the key (see Design notes);
- **there is no API to change a mirror's credentials** (Forgejo 15
  `EditRepoOption` has only `mirror_interval` / `enable_prune`), and Gickup
  never updates an existing mirror's credentials (it only triggers a sync).
  Don't hand-edit the token into the git config either: the DB copy is the
  one Forgejo manages, so a config edit would leave the two out of step.

**Rotation = re-create the mirrors:**

1. Re-seal `gickup-tokens` with the new PAT, merge, let Flux apply it.
2. Take a restore point: `velero backup create --from-schedule forgejo-daily`.
3. Delete the affected mirrors. **Only delete repos that still exist on
   GitHub.** A mirror whose source is gone is the only live copy, and Gickup
   won't re-create it. One at a time: repo → Settings → Danger zone → Delete
   (as `forgejo-admin` or `fqazzazee`).
4. `kubectl -n forgejo create job --from=cronjob/gickup gickup-rotate` and
   check that it exits 0. It re-clones everything (~300MB, a minute or two).

(Alternatively, the per-repo UI at Settings → Repository → Mirror settings →
Authorization updates the stored credential in place. That's 39 forms.)

### Run it now / check it

```bash
kubectl -n forgejo create job --from=cronjob/gickup gickup-manual
kubectl -n forgejo logs -f job/gickup-manual
kubectl -n forgejo get job gickup-manual -o jsonpath='{.status.succeeded}{"\n"}'
kubectl -n forgejo delete job gickup-manual
```

---

## Drift check (`mirror-check`)

`08-mirror-check.yaml`: a daily CronJob at 03:15 UTC (before gickup and the
backup) that compares **every branch and tag** (name → object ID) of every
Forgejo mirror with GitHub, using the same `gickup-tokens`.

**Why not `mirror_updated`:** Forgejo bumps it on *failed* syncs too.
`SyncPullMirror` calls `TouchMirror` when `runSync` fails, and that sets the
same `updated_unix` column the API reports (checked in the v15.0.11 source).
A mirror with a dead PAT therefore looks freshly synced every hour, so only a
content comparison catches it.

| Result | Meaning | Fails the Job? |
|---|---|---|
| in sync | all branches and tags identical | no |
| `pending` | differs, but GitHub saw a push in the last 3h (`GRACE_HOURS`), or the repo is <26h old and gickup hasn't run yet | no |
| `STALE` | differs and the last GitHub push is older than 3h: sync is broken (PAT expired/revoked, GitHub unreachable) | **yes** |
| `MISSING` | on GitHub for >26h, no Forgejo repo: gickup is failing | **yes** |
| `GONE` | Forgejo mirror whose GitHub repo is no longer visible (deleted, renamed, transferred, or the PAT lost access). **The mirror is now the only copy** | **yes** |
| `ERROR` | GitHub/Forgejo API failure (e.g. 401 from an expired PAT) | **yes** |

A failed Job fires the stock `KubeJobFailed` alert. The details are in the log:

```bash
kubectl -n forgejo logs job/$(kubectl -n forgejo get jobs -l app.kubernetes.io/name=mirror-check \
  --sort-by=.metadata.creationTimestamp -o name | tail -1 | cut -d/ -f2)
kubectl -n forgejo create job --from=cronjob/mirror-check mirror-check-manual   # run now
```

**Acknowledging a `GONE` repo:** if the GitHub repo was deleted on purpose,
open the Forgejo repo → Settings → "Convert to regular repository". It stops
being a mirror (and drops out of the check) but keeps all its history. If the
repo was *renamed or transferred*, delete the old mirror after gickup has
created the new one.

Not covered: wiki contents (no refs API) and `refs/pull/*` (mirrored, not
compared).

Tested 2026-11-16 against the live instance: 39/39 in sync in ~26s; injected
faults gave `STALE` (one tag rewritten), `pending` (same drift with a fresh
push), `MISSING`, `GONE` and `ERROR` (401), all with the expected exit codes.
Also ran as uid 65534 with a read-only rootfs in the pinned image.

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
Mirror freshness is `mirror-check`'s job (above), not the weekly report's.

---

## Restore

**One repo to an earlier state** (e.g. after an upstream force-push):
restore into a scratch namespace, clone from there, push wherever needed.
This is the exact procedure tested on 2026-11-16 (see restore-drills.md).

```bash
velero restore create forgejo-scratch-$(date +%Y%m%d) --from-backup <backup> \
  --namespace-mappings forgejo:forgejo-restore \
  --exclude-resources ingresses.networking.k8s.io,certificates.cert-manager.io,cronjobs.batch,jobs.batch,sealedsecrets.bitnami.com \
  --wait
kubectl -n forgejo-restore wait --for=condition=ready pod -l app.kubernetes.io/name=forgejo --timeout=300s
kubectl -n forgejo-restore port-forward deploy/forgejo 3001:3000
git clone http://localhost:3001/fqazzazee/<repo>.git   # log in as forgejo-admin

# clean up (the PV has reclaimPolicy Delete; the Longhorn volume goes with it)
kubectl delete ns forgejo-restore
velero restore delete forgejo-scratch-<date> --confirm
```

Why each exclude: the **Ingress** and **Certificate** would compete with the
live instance for `git.local.safeqbit.com`; the **CronJobs/Jobs** would run a
second gickup and mirror-check; the **SealedSecret** is bound to the `forgejo`
namespace and can't decrypt in another one (the plain `gickup-tokens`
Secret is still restored). The one restore warning to expect is "could not
restore, CustomResourceDefinition … already exists", which is harmless.

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

---

## Change log

| Date | Change |
|---|---|
| 2026-11-15 | Deployed (PR #184). Gickup mirroring (PR #185). |
| 2026-11-16 | Admin group switched from a dedicated `Forgejo Admins` group to the existing `authentik Admins`, set live with `forgejo admin auth update-oauth --id 1 --group-claim-name groups --admin-group "authentik Admins"` (the CLI only changes the flags passed; client ID/secret untouched). GitHub PATs re-issued with 1-year expiry: **both expire 2027-11-16/17**. Rotate before then (see Mirroring). |
| 2026-11-16 | First Gickup run: 39 mirrors (32 + 7), all 209 branches/tags identical to GitHub, fsck clean. First backup (295 MB) + scratch-namespace restore test passed, incl. a mirror sync of a private repo (proves the restored `SECRET_KEY` decrypts the restored DB). `fqazzazee` added to the Zonduit Owners team (setup step 6). `mirror-check` CronJob added instead of the planned `backup-summary-report` check, because `mirror_updated` can't detect failures (see Drift check). |
