# qbit-labs

Self-hosted study platform: lab pages in Markdown, quizzes, spaced-repetition
flashcards, step-through network maps, progress tracking. Added 2026-10-26.

- **Namespace:** `qbit-labs`
- **Hostname:** https://qbit-labs.local.safeqbit.com (LAN only; there is no login, see [Security posture](#security-posture))
- **Manifests:** `apps/safeqbit-local-hq/qbit-labs/`
- **Upstream:** https://github.com/fqazzazee/qbit-labs (my own, **private**)
- **Image:** none of its own. `node:24.22.1-alpine3.24` builds and runs the source at a pinned commit
- **Version pin:** `QBIT_REF` in `04-deployment.yaml`. Commit `3b6b831` = Phase 3 complete (AZ-700)
- **Storage:** `qbit-labs-data` 1Gi Longhorn RWO at `/data`, holding one SQLite file (`qbit.db`, WAL mode)
- **Backup:** `infrastructure/.../velero-schedule-qbit-labs.yaml`, daily 03:45 UTC to B2, 14d retention

---

## How the pod starts

Same idea as [babytracker](babytracker.md) (no image or registry; upstream is the
source of truth), with two extra steps:

| Container | Image | Does |
|---|---|---|
| `fetch-source` (init) | `alpine/git` | Fetches the one pinned commit over SSH with the deploy key, into an emptyDir. Then it deletes `.git` and the key copy |
| `build` (init) | `node:24-alpine` | `npm ci --ignore-scripts` → `npm run build` → `npm prune --omit=dev`. Takes about 10s; peak memory ~1.2GiB, so the limit is 2Gi |
| `qbit-labs` | `node:24-alpine` | `node build`, with source mounted read-only and `/data` on the PVC |

Lab content is in the same repo (`labs/`), so **publishing new pages = an
upgrade** (bump the SHA). The PVC holds progress only.

What this costs:

- **Every pod start needs github.com:22 and registry.npmjs.org.** A running pod
  needs neither. `dnsConfig ndots:1` is set because musl resolvers otherwise try
  `*.local.safeqbit.com` first and can hard-fail
  (see the DNS search-amplification note in maintenance.md).
- **Start time is ~30s** instead of ~3s.
- **`npm ci` reproduces `package-lock.json` exactly**, with integrity checks, and
  `--ignore-scripts` means no dependency install hook ever runs in the cluster.
  If a future version needs a native addon's postinstall, the build fails loudly
  rather than silently. (Upstream's DECISIONS.md forbids native addons anyway.)

Watch a start:

```sh
kubectl -n qbit-labs get pod -w
kubectl -n qbit-labs logs deploy/qbit-labs -c fetch-source   # "checked out <sha>"
kubectl -n qbit-labs logs deploy/qbit-labs -c build          # "built 11M, node_modules 212M"
kubectl -n qbit-labs logs deploy/qbit-labs                   # "Listening on http://0.0.0.0:3000"
```

## The deploy key

The repo is private, so the clone needs a credential. It is a **read-only SSH
deploy key** on that one repo, rather than a PAT: it can't read any other repo,
can't push, and doesn't expire. GitHub shows it as
`k3s-safeqbit-local-hq (read-only)` under the repo's Settings → Deploy keys. The
private half exists only in `03-deploy-key.sealed.yaml`. It was generated in a
temp dir that was deleted after sealing.

GitHub's host keys are pinned in the manifest (`GITHUB_KNOWN_HOSTS`, from
`https://api.github.com/meta`) with `StrictHostKeyChecking=yes`. If GitHub ever
rotates its host key, `fetch-source` fails with `Host key verification failed`;
update the env var from that URL.

`fetch-source` mounts a two-line `/etc/passwd` (ConfigMap `qbit-labs-ssh-passwd`)
over the image's own copy. OpenSSH refuses to run as a uid with no passwd entry
(`No user exists for uid 1000`), and `alpine/git` has none for 1000. Podman
injects one, so a local test won't catch this.

**Re-key / rotate:**

```sh
# 1. delete the old key in GitHub (Settings → Deploy keys), or:
gh repo deploy-key list -R fqazzazee/qbit-labs
gh repo deploy-key delete <id> -R fqazzazee/qbit-labs
# 2. make + seal a new one (writes 03-deploy-key.sealed.yaml)
K=$(mktemp -d); ssh-keygen -q -t ed25519 -N '' -f $K/id
gh repo deploy-key add $K/id.pub -R fqazzazee/qbit-labs -t 'k3s-safeqbit-local-hq (read-only)'
kubectl create secret generic qbit-labs-deploy-key -n qbit-labs \
  --from-file=ssh-privatekey=$K/id --dry-run=client -o yaml \
  | kubeseal --controller-namespace kube-system --controller-name sealed-secrets-controller --format yaml \
  > apps/safeqbit-local-hq/qbit-labs/03-deploy-key.sealed.yaml   # then re-add the header comment
rm -rf $K
```

A dead key only matters at the next pod start: a running pod keeps serving.

## Upgrading

1. Read the upstream diff. Watch for **schema changes in
   `src/lib/server/progress.ts`**. The SQLite file migrates forward, and a
   rollback onto an older schema may not read it. Also watch the Node
   `engines` field (the image is Node 24) and new runtime `dependencies`.
2. Back up first if the schema moves:
   ```sh
   kubectl -n velero exec deploy/velero -- \
     /velero backup create qbit-labs-manual-$(date +%Y%m%d) --from-schedule qbit-labs-daily
   ```
3. Change `QBIT_REF` **and** the `qbit-labs.safeqbit.com/source-ref` annotation
   to the new full SHA (`git rev-parse HEAD` in the qbit-labs repo).
4. Commit, push, `flux reconcile kustomization apps --with-source`.

Content-only bumps (new lab pages) need none of the care in steps 1–2.

**Dry run** before anything risky. This is the same build the pod does, in podman,
against a copy of the live DB:

```sh
git -C ~/git/qbit-labs archive --prefix=src/ <sha> | tar -x -C /tmp/qb/app
podman run --rm -u 1000 --userns=keep-id -v /tmp/qb/app:/app:Z -e HOME=/tmp -w /app/src \
  node:24.22.1-alpine3.24 sh -c 'npm ci --ignore-scripts && npm run build && npm prune --omit=dev'
```

## Security posture

qbit-labs v1 has **no authentication**. Upstream says so in its README and in
DECISIONS.md: anyone who can reach it can read the labs and change progress.
That's acceptable on the LAN, so:

- internal hostname only, with no cloudflared tunnel, no Pangolin resource, and no public record;
- if a login is ever wanted before upstream adds one, use the Authentik forward-auth
  pattern from prometheus/alertmanager (maintenance.md, "Authentik forward auth").
  The auth-url must hairpin through the app hostname.

The data is low sensitivity (study progress and bookmark notes).

## Single replica

SQLite on an RWO volume, so one writer. `Recreate` strategy, 60s node-down
tolerations, same as the other single-writer apps. Don't scale it.
