# Home Assistant migration — Docker host → k3s

**Status:** manifests merged, data cutover pending
**Namespace:** `home-assistant` · **Hostname:** `ha.local.safeqbit.com` · **Image:** `ghcr.io/home-assistant/home-assistant:2026.7.1`

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

### 0. Pre-flight (Docker host)

```sh
# Version check — the k3s image tag must be >= this (HA migrates the DB
# schema forward on boot and CANNOT downgrade):
docker exec <ha-container> cat /config/.HA_VERSION

# Find the /config bind mount on the host:
docker inspect <ha-container> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
```

### 1. Let Flux deploy the app

Merge the manifests; Flux creates namespace/PVC/Deployment/Ingress. The pod
boots with an empty `/config` and shows the onboarding screen — expected,
we overwrite it in step 3. Verify `https://ha.local.safeqbit.com` serves the
onboarding page (proves ingress + cert + PVC attach) but **do not complete
onboarding**.

### 2. Freeze both sides

```sh
flux suspend kustomization apps
kubectl scale deploy -n home-assistant home-assistant --replicas=0

# Docker host — final state snapshot (graceful stop = restore-state saved):
docker stop <ha-container>
tar czf ha-config.tgz -C /path/to/ha-config .
scp <docker-host>:ha-config.tgz .
```

### 3. Fix the config INSIDE the tarball's source, or after extraction

Three edits, all in `configuration.yaml` unless noted — doing them now means
first boot in k3s is already correct:

1. **Trusted proxies** (mandatory — HA returns 400 behind ingress-nginx
   without it):

   ```yaml
   http:
     use_x_forwarded_for: true
     trusted_proxies:
       - 10.42.0.0/16   # k3s pod CIDR (verified on cluster)
   ```

2. **Packages enabled + baby comfort sensors** (see
   `local-only/home-assistant/`): add under the existing `homeassistant:`
   block (or create one — never a duplicate key):

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

   and copy `local-only/home-assistant/packages/baby_comfort.yaml` into
   `packages/`.

3. **Delete every `initial:` line** under `input_boolean:` / `input_number:`
   / `input_select:` / `input_datetime:` — `initial:` re-applies on every
   restart and is why sliders/toggles were resetting on the Docker host.
   Without it HA restores last values from `.storage/core.restore_state`.

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
kubectl exec -i -n home-assistant ha-copy -- tar xzf - -C /config < ha-config.tgz
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

- [ ] Login works at `ha.local.safeqbit.com` with existing credentials
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
      app's server URL, to `https://ha.local.safeqbit.com`
- [ ] Paste `local-only/home-assistant/dashboard-apartment.yaml` via
      dashboard ⋮ → Raw configuration editor (adds Baby Comfort section)

### 6. Decommission (after ~1 week soak)

Keep the Docker container **stopped but present** + the tarball as rollback.
Rollback = `docker start <ha-container>` (and scale the k3s deploy to 0 —
two live instances would fight over cloud sessions). After soak: remove the
container, keep the tarball until the first successful Velero backup
(`home-assistant-weekly`, Sundays 06:00 UTC, 60d TTL) is verified.

## Gotchas learned / to remember

- **Never lower the image tag** once a version has booted against the PVC —
  recorder schema migrations are one-way.
- Restarts: normal `kubectl delete pod` is fine (Recreate + 120s
  terminationGracePeriod = graceful shutdown = restore-state saved).
- `initial:` on YAML helpers overrides restore-state every boot — that was
  the Docker-era "sliders reset" bug. Don't reintroduce it.
- mDNS/zeroconf/SSDP discovery is dead on the pod network. Existing
  integrations keep working (config is in `.storage`); *new* discovery-based
  devices must be added by IP.
