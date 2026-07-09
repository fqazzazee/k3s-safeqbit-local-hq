# Home Assistant migration — Docker host → k3s

**Status:** CUTOVER DONE 2026-07-09 (clean boot, 0 errors, cert issued) — soak until ~2026-07-16, then decommission the Docker container
**Namespace:** `home-assistant` · **Hostname:** `home-assistant-01.local.safeqbit.com` · **Image:** `ghcr.io/home-assistant/home-assistant:2026.4.1` (= exact Docker-host version; lift-and-shift)

**Upgrade gate (2026-07-09):** Sensi is a HACS **custom component**
(`custom_components/sensi`, iprak/sensi v2.1.4), plus HACS itself. Core was
deliberately NOT bumped during migration (host ran 2026.4.1). To upgrade core
later: check iprak/sensi compatibility → bump the image tag → update HACS
components. Never downgrade the tag once booted.

## Why / shape

HA Container's entire state is `/config`: integration credentials + entity
registry (`.storage/`), `configuration.yaml`, automations, dashboards, helper
values (restore state), and the recorder SQLite DB. No external database, so
this is the simplest migration class we have: one PVC, copy the directory,
done. Same single-writer doctrine as uptime-kuma/pulse — **1 replica +
Recreate, never scale**; resilience = 60s tolerations + Longhorn
node-down-pod-deletion-policy.

All devices reach HA over network/cloud (Sensi = cloud, UniFi Protect =
direct IP, HTG3 = network/cloud, TV = LAN) — no USB/Bluetooth passthrough,
no hostNetwork needed. Confirmed 2026-07-09.

Built-in HA "backups" are just tars of `/config`; on a Container install,
restoring one onto a new instance means extracting it into `/config` anyway —
so we do the direct equivalent: stop container → tar → untar onto PVC.

## Cutover procedure

### 0. Pre-flight — DONE 2026-07-09

Host `.HA_VERSION` = **2026.4.1** (frontend 20260325.6); image pinned to
match. Config tarred and staged on the workstation at
`local-only/home-assistant/` (git-ignored):

- `ha-config.tar` — raw snapshot from the host (216M, wraps a `config/`
  prefix; taken while HA was running — `-wal`/`-shm` present)
- `ha-config-ready.tgz` — extracted, **fixes applied**, re-packed
  root-relative (stream straight into `/config`), logs excluded
- `fix-config.sh` — the fixes as an idempotent script (see step 3)

### 1. Let Flux deploy the app

Merge the manifests; Flux creates namespace/PVC/Deployment/Ingress. The pod
boots with an empty `/config` and shows the onboarding screen — expected,
we overwrite it in step 3. Verify `https://home-assistant-01.local.safeqbit.com` serves the
onboarding page (proves ingress + cert + PVC attach) but **do not complete
onboarding**.

### 2. Freeze both sides

```sh
flux suspend kustomization apps
kubectl scale deploy -n home-assistant home-assistant --replicas=0

# Docker host — final state snapshot (graceful stop = restore-state saved):
docker stop <ha-container>
```

The staged `ha-config-ready.tgz` was tarred while HA was running — fine for
everything except possibly the last few hours of recorder history (SQLite
WAL not checkpointed; worst case HA moves a corrupt DB aside and starts
fresh history — config/credentials unaffected). For a byte-perfect cutover,
re-tar after the stop and re-apply fixes:

```sh
tar czf ha-config-fresh.tgz -C /path/to/ha-config .   # on the Docker host
scp <docker-host>:ha-config-fresh.tgz . && mkdir fresh && tar xzf ha-config-fresh.tgz -C fresh
sh local-only/home-assistant/fix-config.sh fresh
tar czf ha-config-ready.tgz -C fresh --exclude='home-assistant.log*' .
```

### 3. Config fixes — DONE (baked into `ha-config-ready.tgz`)

Applied by `local-only/home-assistant/fix-config.sh` (idempotent; re-run it
if step 2 produced a fresh tar):

1. **Trusted proxies**: `10.42.0.0/16` (k3s pod CIDR, verified) added to the
   existing `http.trusted_proxies` list; old Docker-era proxy entries kept
   (harmless). Without this HA 400s behind ingress-nginx.
2. **Baby comfort sensors**: `baby_comfort.yaml` copied into `packages/`
   (packages were already enabled — the whole smart-climate system lives in
   `packages/smart_climate.yaml`).
3. **Sliders-reset bug**: all 23 `initial:` lines deleted from
   `packages/smart_climate.yaml` — `initial:` re-applies on every restart,
   overriding restore-state. Without it HA restores last helper values from
   `.storage/core.restore_state`.

### 4. Copy data onto the PVC

```sh
# Helper pod holding the PVC (deployment is scaled to 0, so RWO is free):
kubectl run -n home-assistant ha-copy --restart=Never \
  --image=alpine:3.20 --overrides='{"spec":{"containers":[{"name":"ha-copy",
  "image":"alpine:3.20","command":["sleep","7200"],
  "volumeMounts":[{"name":"config","mountPath":"/config"}]}],
  "volumes":[{"name":"config","persistentVolumeClaim":
  {"claimName":"home-assistant-config"}}]}}'

# Wipe what the empty first boot wrote (incl. dotfiles), then stream in:
kubectl exec -n home-assistant ha-copy -- sh -c 'rm -rf /config/* /config/.[!.]* 2>/dev/null; true'
kubectl exec -i -n home-assistant ha-copy -- tar xzf - -C /config < local-only/home-assistant/ha-config-ready.tgz
kubectl exec -n home-assistant ha-copy -- cat /config/.HA_VERSION   # sanity
kubectl delete pod -n home-assistant ha-copy
```

### 5. Start and verify

```sh
kubectl scale deploy -n home-assistant home-assistant --replicas=1
flux resume kustomization apps
kubectl logs -n home-assistant deploy/home-assistant -f
```

Checklist:

- [ ] Login works at `home-assistant-01.local.safeqbit.com` with existing credentials
- [ ] Sensi thermostat online (cloud tokens carry over in `.storage`)
- [ ] UniFi Protect camera streams (direct IP)
- [ ] HTG3 room sensors updating
- [ ] `sensor.baby_comfort_*` + `sensor.baby_best_room` exist (packages loaded)
- [ ] Flip a slider, `kubectl delete pod`, confirm value survives (the
      original bug, now fixed)
- [ ] **fofo TV**: zeroconf/mDNS doesn't cross the pod network — if the cast
      integration shows it unavailable, set the TV's IP in the integration's
      options ("known hosts"). Only integration expected to need hand-holding.
- [ ] Update Settings → System → Network → internal URL, and the mobile
      app's server URL, to `https://home-assistant-01.local.safeqbit.com`
- [ ] Paste `local-only/home-assistant/dashboard-apartment.yaml` via
      dashboard ⋮ → Raw configuration editor (adds Baby Comfort section)

### 6. Decommission (after ~1 week soak)

Keep the Docker container **stopped but present** + the tarball as rollback.
Rollback = `docker start <ha-container>` (and scale the k3s deploy to 0 —
two live instances would fight over cloud sessions). After soak: remove the
container, keep the tarball until the first successful Velero backup
(`home-assistant-biweekly`, 1st/15th/29th 06:00 UTC, 28d TTL ≈ keep last 2)
is verified. Since the next scheduled run may be days out, trigger one
manually after the soak: exec in the velero pod →
`velero backup create --from-schedule home-assistant-biweekly`.

## Gotchas learned / to remember

- **ndots:1 is load-bearing (found during cutover, PR #49):** with the
  default ndots:5, the HA image's musl resolver hard-fails ALL external
  lookups via the `local.safeqbit.com` search-domain amplification (upstream
  answers NOERROR/no-answer, musl stops). Broke Sensi (pip-installs
  python-socketio at boot → "Setup failed") AND Jellyfin. Same fix as the
  slack bot. Also: live `kubectl patch` does not survive — flux-system
  re-applies apps.yaml (clears `suspend:`) and SSA strips out-of-Git fields;
  such fixes must land via Git.
- **Never lower the image tag** once a version has booted against the PVC —
  recorder schema migrations are one-way.
- Restarts: normal `kubectl delete pod` is fine (Recreate + 120s
  terminationGracePeriod = graceful shutdown = restore-state saved).
- `initial:` on YAML helpers overrides restore-state every boot — that was
  the Docker-era "sliders reset" bug. Don't reintroduce it.
- mDNS/zeroconf/SSDP discovery is dead on the pod network. Existing
  integrations keep working (config is in `.storage`); *new* discovery-based
  devices must be added by IP.
