# Home Assistant config — versioned in Git, applied by hand

Source of truth for the hand-maintained pieces of the HA `/config` (the rest
of `/config` lives on the `home-assistant-config` Longhorn PVC and is backed
up by Velero). **Nothing in this directory is applied by Flux** — the app's
`kustomization.yaml` lists its resources explicitly, so these files are
ignored by kustomize. Applying is always a manual step:

| File | Apply with |
|---|---|
| `dashboard-apartment.yaml` | Dashboard ⋮ → **Raw configuration editor** → paste → save |
| `packages/*.yaml` | `kubectl -n home-assistant cp <file> <pod>:/config/packages/` then Developer Tools → YAML → **Reload Template Entities** |

```sh
POD=$(kubectl -n home-assistant get pod -o jsonpath='{.items[0].metadata.name}')
kubectl -n home-assistant cp packages/baby_comfort.yaml     "$POD":/config/packages/baby_comfort.yaml
kubectl -n home-assistant cp packages/weather_insights.yaml "$POD":/config/packages/weather_insights.yaml
```

## Versioning

Every file carries a `# Version: X.Y — YYYY-MM-DD` header with a short
changelog. Bump the version and add a line whenever you change a file, and
apply the same edit to the live HA (the header makes drift detectable:
compare the version comment in Git against what's pasted/copied into HA).

## Contents

- `dashboard-apartment.yaml` — the Apartment dashboard (Overview / Smart
  Climate Controls / Rooms / Security Cameras). Requires both packages below.
- `packages/baby_comfort.yaml` — per-room newborn comfort ratings,
  0–100 % comfort scores (gauge scale), multi-room "comfy rooms" picker.
- `packages/weather_insights.yaml` — trigger-based forecast pull (Met.no via
  `weather.get_forecasts`, every 15 min): storm watch (24 h), rain watch
  (12 h), nice-day verdict with reason.

`packages/smart_climate.yaml` is NOT here — it predates the migration and
lives only in `/config` on the PVC (fixed at cutover by
`local-only/home-assistant/fix-config.sh`). Pull it in with a version header
the next time it changes.
