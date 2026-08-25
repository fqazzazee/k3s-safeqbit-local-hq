# Ultimate Baby Tracker

One-tap newborn tracking — feeds, diapers, sleep, custom buttons, programmable
alarms. Added 2026-08-25.

- **Namespace:** `babytracker`
- **Hostname:** https://babytracker.local.safeqbit.com (internal only — see [Security posture](#security-posture))
- **Manifests:** `apps/safeqbit-local-hq/babytracker/`
- **Upstream:** https://github.com/fqazzazee/ultimate-baby-tracker (my own, MIT)
- **Image:** none of its own — `node:22.23.2-alpine3.24` running source cloned at a pinned commit (see [Why there is no image](#why-there-is-no-image))
- **Version pin:** `BT_REF` in `03-deployment.yaml` — commit `047d682` = v1.1.1
- **Storage:** `babytracker-data` 1Gi Longhorn RWO at `/data` — plain-text JSON, the *only* copy of the log
- **Backup:** `infrastructure/.../velero-schedule-babytracker.yaml` — daily 03:30 UTC to B2, 30d retention

---

## Deploy

Flux, like every other app — no manual `kubectl apply`.

```sh
git add apps/safeqbit-local-hq/babytracker \
        apps/safeqbit-local-hq/kustomization.yaml \
        infrastructure/safeqbit-local-hq/configs/velero-schedule-babytracker.yaml \
        infrastructure/safeqbit-local-hq/configs/kustomization.yaml
git commit -m "Add Ultimate Baby Tracker"
git push
flux reconcile kustomization apps --with-source
```

**DNS is the one manual step:** add `babytracker.local.safeqbit.com` as a CNAME
to `ingress.local.safeqbit.com` on the router, same as every other app. The
certificate does *not* depend on it — cert-manager proves control of the
`safeqbit.com` zone through the Cloudflare API (DNS-01), not by being reachable.

Watch it come up:

```sh
kubectl -n babytracker get pod -w
kubectl -n babytracker logs deploy/babytracker -c fetch-source   # "checked out <sha>"
kubectl -n babytracker logs deploy/babytracker                   # "[store] /data - N events…"
kubectl -n babytracker get certificate                           # Ready=True within ~2 min
```

## Why there is no image

The app publishes no container image, and this cluster has no registry to push
one to. It needs neither: it is zero-dependency ESM JavaScript — no build step,
no `npm install` — so a stock Node image plus the source tree *is* the whole
deployment.

So the pod has an initContainer (`alpine/git`) that fetches **one pinned commit**
into an emptyDir, and the Node container runs it from there read-only. Upstream
stays the single source of truth and an upgrade is a one-line commit here.

What this costs, stated plainly:

- **A pod start needs egress to github.com.** A pod that is already running does
  not, so a GitHub outage can't take the tracker down — it can only delay a
  restart.
- **The upstream repo must stay public.** It was private until 2026-08-25; an
  anonymous clone is what makes this work without a token in the cluster. If it
  ever goes private again, the pod will fail to start with
  `could not read Username for 'https://github.com'`, and the fix is a
  fine-grained read-only PAT sealed into the namespace and injected into the
  clone URL — not a lot of work, but it is work.
- **Pinned to a full SHA, never a branch**, so two pods started a week apart run
  byte-identical code and `main` moving can't restart you onto a new on-disk
  format.

## Upgrading

1. Read the upstream diff. `events.log` is replayed into memory at startup, so a
   change to how entries are written is a **one-way door** for existing data.
2. Back up first:
   ```sh
   kubectl -n velero exec deploy/velero -- \
     /velero backup create babytracker-manual-$(date +%Y%m%d) --from-schedule babytracker-daily
   ```
   (`kubectl -n velero get backups.velero.io` — the bare `get backup` in that
   namespace resolves to `backups.longhorn.io`.)
3. Change `BT_REF` **and** the `babytracker.safeqbit.com/source-ref` pod
   annotation to the new SHA. They are the same value in two places on purpose:
   the annotation is what makes `kubectl describe pod` answer "which code is
   this running".
4. Commit, push, `flux reconcile kustomization apps --with-source`.

Rollback is the same edit with the old SHA — the code is stateless, only `/data`
carries forward.

## Security posture

**The app has no authentication and no authorization of any kind.** Every HTTP
endpoint answers anyone who can reach it; the whole log is readable *and*
writable unauthenticated. Upstream says so in as many words, and the deployment
is built around that fact rather than in spite of it:

- Internal hostname only. **Do not** point a cloudflared tunnel at this Service
  (that is what makes passzilla public — don't copy that pattern here) and don't
  add a record for it in the public `safeqbit.com` zone.
- The 4-digit profile PINs gate *switching users in the UI* and nothing else.
  They stop entries being logged under the wrong name between people who already
  trust each other. They are not a control on the API.
- If it ever needs to be reachable from outside the LAN, put Authentik forward
  auth in front of it first — the embedded-outpost pattern in
  `maintenance.md` ("Authentik forward auth"), with the auth-url hairpinned
  through the app hostname.

The pod itself is locked down as far as the app allows: non-root uid 1000,
read-only root filesystem, all capabilities dropped, no service account tokens
in play. The only writable paths are `/data` (the PVC), `/tmp`, and the emptyDir
holding the source.

## Single replica, permanently

The store is an append-only file replayed into memory at boot, and the SSE
stream broadcasts from that same in-memory state. Two replicas would be two
divergent in-memory copies writing one log — on an RWO Longhorn volume that
won't attach twice anyway. Same single-writer doctrine as vaultwarden /
uptime-kuma / home-assistant: resilience is reschedule speed (60s tolerations +
Longhorn `nodeDownPodDeletionPolicy`), not a second replica. Don't re-litigate
this one.

## Live updates (SSE)

`/api/stream` is a server-sent-events connection the UI holds open forever;
every logged entry pushes down it, which is how a phone in another room updates
instantly. Three things keep it alive and all three are already in place:

- the app sets `x-accel-buffering: no` on the response, so nginx doesn't buffer it,
- it heartbeats a `: ping` comment every 25s,
- the Ingress sets `proxy-read-timeout: 3600` and `proxy-buffering: off`
  explicitly, so a change to the ingress-nginx defaults can't quietly break it.

Symptom if this regresses: the UI still works but a tap on one phone takes until
a manual refresh to show on another.

## What lives in `/data`

```
config.json    babies, people, buttons, alarms, settings (pretty JSON)
events.log     one JSON object per line, append-only journal
timers.json    timers currently running (a running timer survives a restart)
alarms.json    snooze / last-fired state
```

`events.log` is a journal: edits and deletions are appended as further lines and
the file is replayed at startup, so a crash can't corrupt earlier entries. It
compacts itself once tombstones pile up. All of it is readable text — `kubectl
-n babytracker exec deploy/babytracker -- cat /data/events.log` is a legitimate
debugging move, and copying the folder is a legitimate backup.

## Troubleshooting

**Pod stuck in `Init:0/1`** — check `kubectl -n babytracker logs deploy/babytracker
-c fetch-source`. `could not read Username for 'https://github.com'` means the
upstream repo went private (see [above](#why-there-is-no-image)). A network
error means egress to github.com is broken; the running pod was fine until this
restart, so nothing is lost by waiting.

**`CrashLoopBackOff` right after an upgrade** — almost certainly the new code
choking on the existing `events.log`. Put `BT_REF` back to the old SHA; the data
is untouched.

**Alarms fire at the wrong hour** — `TZ` on the container. Note that
`kubectl exec … date` prints **UTC** in this pod no matter what `TZ` says,
because alpine ships no `/usr/share/zoneinfo`; that is a red herring. Node reads
its own bundled ICU tzdata, so ask Node instead:

```sh
kubectl -n babytracker exec deploy/babytracker -- node -e 'console.log(new Date().toString())'
```

**Everything gone / fresh install screen** — the PVC didn't attach, or attached
empty. Check `kubectl -n babytracker get pvc` and the Longhorn volume before
touching anything; do **not** start re-entering data into a pod that has mounted
the wrong volume. Restore below.

## Restore (DR)

```sh
kubectl -n velero get backups.velero.io | grep babytracker
kubectl -n velero exec deploy/velero -- \
  /velero restore create --from-backup babytracker-daily-<timestamp> \
    --include-namespaces babytracker
```

Restored `CertificateRequest`/`Order`/`Challenge` objects are excluded from the
schedule on purpose — restoring them plants a stale request that silently stalls
the next renewal while the certificate still reads `Ready`.

To pull just the log out of a backup without a full restore, restore into a
scratch namespace and `kubectl cp` the files off the pod.
