# Cloudflared: GitOps Tunnel Migration

**Cluster:** safeqbit-local-hq  
**Completed:** 2026-05-21  
**Migrated from:** standalone Docker host (each tunnel ran as `docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <TOKEN>`)

---

## Current State

```
Namespace: cloudflared
├── tunnel-passzilla              (2 connector pods, token sealed)
├── tunnel-ghost-blog             (2 connector pods, token sealed)
├── tunnel-csra                   (2 connector pods, token sealed)
└── tunnel-family-media-server    (2 connector pods, token sealed)

Each tunnel has:
  ├── SealedSecret   → cloudflared-<name>-cf-tunnel-token
  ├── Deployment     → 2 replicas, spread across nodes
  ├── Service        → headless, metrics port 2000 only
  ├── ServiceMonitor → Prometheus scrapes metrics every 30s
  └── PodDisruptionBudget → minAvailable: 1
```

Routing rules (hostname → origin service) live in the **Cloudflare Zero Trust dashboard**, not in the repo. cloudflared pulls them remotely via the management API.

---

## Architecture

```
Internet → Cloudflare Edge → tunnel connector pods → k8s ClusterIP service → app pods
```

- Tunnels are **outbound-only**. Connector pods dial Cloudflare's edge (QUIC/HTTP2). No inbound ports or LoadBalancer services needed for traffic.
- Two replicas per tunnel = two connectors. Cloudflare load-balances across them automatically. Both use the same token.
- Token is remote-managed: the routing config is pushed from the Cloudflare dashboard to the running connector, no local config file needed.

---

## Repo Layout

```
apps/safeqbit-local-hq/cloudflared/
├── kustomization.yaml
├── 00-namespace.yaml
├── prometheusrule.yaml          ← shared alerts for all tunnels (by namespace)
├── tunnel-passzilla.yaml        ← SealedSecret + Deployment + Service + ServiceMonitor + PDB
├── tunnel-ghost-blog.yaml
├── tunnel-csra.yaml
└── tunnel-family-media-server.yaml
```

---

## Adding a New Tunnel

### 1 — Get the token from Cloudflare Zero Trust dashboard

Zero Trust → Tunnels → \<your tunnel\> → Configure → copy the token from the `docker run` command.

### 2 — Seal the token

```bash
kubectl create secret generic <name>-cf-tunnel-token \
  --namespace cloudflared \
  --from-literal=token='<PASTE_TOKEN_HERE>' \
  --dry-run=client -o yaml \
| kubeseal \
    --controller-name  sealed-secrets-controller \
    --controller-namespace kube-system \
    --format yaml \
> /tmp/<name>-sealed.yaml

# Copy just the encrypted value
grep "token:" /tmp/<name>-sealed.yaml
```

### 3 — Create the tunnel YAML

```bash
cp apps/safeqbit-local-hq/cloudflared/tunnel-passzilla.yaml \
   apps/safeqbit-local-hq/cloudflared/tunnel-<name>.yaml

sed -i 's/passzilla-cf-tunnel/<name>-cf-tunnel/g' \
   apps/safeqbit-local-hq/cloudflared/tunnel-<name>.yaml
```

Paste the sealed token value into `spec.encryptedData.token`.

### 4 — Register in kustomization

```yaml
# apps/safeqbit-local-hq/cloudflared/kustomization.yaml
resources:
  - tunnel-<name>.yaml       # add this line
```

### 5 — Configure origin in Cloudflare dashboard

Set the service URL to the **k8s internal service DNS** — not the `*.local.safeqbit.com` hostname:

```
http://<service-name>.<namespace>.svc.cluster.local
```

See below for how to find the right service name and port.

### 6 — Check for stale DNS

After saving the public hostname in the dashboard, verify the DNS CNAME in `safeqbit.com` DNS points to the new tunnel's UUID — not a leftover record from a previous Docker-based tunnel. Delete the old record first if one exists.

---

## Finding the Internal Service URL for a k8s App

When configuring a public hostname in the Cloudflare Zero Trust dashboard, use the app's ClusterIP service, not the ingress hostname. This avoids CNAME chain DNS failures and unnecessary MetalLB roundtrips.

```bash
# List all services across all namespaces — find the app's service name and port
kubectl get svc -A

# Filter to a specific app namespace
kubectl get svc -n <namespace>

# Get the full internal DNS name and port for an app
kubectl get svc -n <namespace> <service-name> \
  -o jsonpath='http://{.metadata.name}.{.metadata.namespace}.svc.cluster.local:{.spec.ports[0].port}'

# Verify cloudflared can actually reach it before configuring the dashboard
kubectl run reach-test --rm -it --restart=Never --image=busybox:1.36 \
  --namespace=cloudflared \
  -- wget -qO- --timeout=5 http://<service>.<namespace>.svc.cluster.local
```

### Current tunnel → service mappings

| Tunnel | Public hostname | Internal service URL |
|---|---|---|
| passzilla | passzilla.safeqbit.com | `http://passzilla.passzilla.svc.cluster.local` |
| ghost-blog | — | `http://<svc>.<ns>.svc.cluster.local` |
| csra | — | `http://<svc>.<ns>.svc.cluster.local` |
| family-media-server | — | `http://<svc>.<ns>.svc.cluster.local` |

---

## Monitoring

### Check tunnel pod health

```bash
# All tunnel pods at a glance
kubectl get pods -n cloudflared -o wide

# Logs for a specific tunnel (shows edge connection status and config updates)
kubectl logs -n cloudflared -l app=<name>-cf-tunnel --since=1h

# Confirm edge connections — should show 4 registered connections per pod
kubectl logs -n cloudflared -l app=<name>-cf-tunnel | grep "Registered tunnel"
```

### Check live metrics

```bash
# Port-forward to the metrics endpoint of one connector pod
kubectl port-forward -n cloudflared deploy/<name>-cf-tunnel 2000:2000

# In another terminal
curl -s http://localhost:2000/metrics | grep -E "^cloudflared_tunnel"
```

Key metrics:

| Metric | What it means |
|---|---|
| `cloudflared_tunnel_ha_connections` | Active connections to Cloudflare edge (healthy = 4) |
| `cloudflared_tunnel_total_requests` | Requests proxied since pod start |
| `cloudflared_tunnel_request_errors` | Failed proxy attempts |
| `cloudflared_tunnel_server_locations` | Which Cloudflare PoP each connector is hitting |

### Alerting

`prometheusrule.yaml` defines four alerts evaluated by Prometheus:

| Alert | Severity | Fires when |
|---|---|---|
| `CloudflaredTunnelDown` | critical | A tunnel Deployment has 0 available replicas for 2m |
| `CloudflaredTunnelDegraded` | warning | Fewer replicas available than desired for 5m |
| `CloudflaredMetricsMissing` | warning | Prometheus cannot scrape a connector pod for 5m |
| `CloudflaredConnectorCrashLooping` | warning | Container restarts > 3 times in 1h |

**Note:** Alertmanager is disabled in this cluster (`alertmanager: enabled: false` in `monitoring.yaml`). Alerts are evaluated and visible in the Prometheus UI but do not send notifications.

---

## Lessons Learned

### Use k8s internal service DNS as the origin, not `*.local.safeqbit.com`

The local domain goes through a CNAME chain (`passzilla.local.safeqbit.com → ingress.local.safeqbit.com → 10.10.13.50`) resolved by the router. Go's DNS resolver in cloudflared (with `ndots:5`) can fail on this chain intermittently. Internal service DNS (`*.svc.cluster.local`) is resolved natively by CoreDNS — no forwarding, no CNAME chain, always stable.

### Clean up old DNS records when migrating a tunnel from Docker

Cloudflare's DNS CNAME for a public hostname (`passzilla.safeqbit.com`) points to `<tunnel-uuid>.cfargotunnel.com`. When the Docker tunnel is shut down and a new k8s tunnel takes over with a different UUID, the old CNAME still points at the now-dead connector. Cloudflare returns **Error 1033** even though the new tunnel is connected and healthy. Fix: delete the old CNAME in Cloudflare DNS before or immediately after creating the public hostname on the new tunnel.

### Error 1033 with zero request logs = Cloudflare-side routing problem, not local

If cloudflared logs show healthy edge connections but `cloudflared_tunnel_total_requests` stays at 0, the issue is upstream of cloudflared (wrong DNS CNAME, wrong tunnel selected in dashboard). Local service URL and DNS are not the problem.

### kubeseal must match controller version

Sealed with kubeseal **v0.36.6** against controller **v0.36.6** (`bitnami/sealed-secrets-controller:0.36.6` in `kube-system`). If the controller is upgraded, re-seal all tokens.
