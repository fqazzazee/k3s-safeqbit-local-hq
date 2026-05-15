# Cluster Maintenance Runbook

**Cluster:** safeqbit-local-hq  
**Last updated:** 2026-05-15

A living document. Add entries as new patterns emerge.

---

## Table of Contents

- [Grafana](#grafana)
- [Flux](#flux)
- [CNPG](#cnpg)
- [Sealed Secrets](#sealed-secrets)
- [General Pod Ops](#general-pod-ops)

---

## Grafana

### Grafana UI is unresponsive / 2/3 containers ready

**Symptom:** Browser returns 502/503, or the dashboard loads but panels spin forever. Pod shows `2/3 Running`.

**Root cause pattern:** Grafana uses Authentik for SSO. If Authentik is unavailable (migration, restart, upgrade) or if a kube-prometheus-stack Helm upgrade touches monitoring ConfigMaps mid-load, the grafana-sc sidecars push updated provisioning files and trigger a plugin reload. Auth requests that land during that window time out, cascade into SQLite `SQLITE_BUSY` locks, and Grafana stops serving requests entirely.

**Confirm it:**
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana --tail=30 \
  | grep -E "ERROR|locked|database|plugin|canceled"
```

Look for: `database is locked (SQLITE_BUSY)`, `plugin not registered`, `context canceled`, liveness probe failures in events.

**Fix — restart the pod:**
```bash
kubectl rollout restart deployment -n monitoring monitoring-grafana
kubectl rollout status deployment -n monitoring monitoring-grafana --timeout=120s
```

Or if the pod is already in a bad state and you want it gone immediately:
```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
kubectl rollout status deployment -n monitoring monitoring-grafana --timeout=120s
```

**When to expect this:** After any kube-prometheus-stack HelmRelease upgrade, or after any Authentik downtime (migration, pod restart, sealed secret rotation).

---

### Check which Grafana container is not ready

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana \
  | grep -A3 "State:\|Ready:\|Last State:"
```

The pod has 3 containers: `grafana` (main), `grafana-sc-dashboard` (sidecar syncing dashboard ConfigMaps), and `grafana-sc-datasources` (sidecar syncing datasource ConfigMaps). It's almost always the main `grafana` container that becomes not-ready.

---

### Force Flux to re-reconcile the monitoring HelmRelease

Useful after pushing a values change and wanting Flux to apply it immediately rather than waiting up to 1 hour:

```bash
kubectl annotate helmrelease monitoring -n monitoring \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --overwrite
```

Check the result:
```bash
kubectl get helmrelease -n monitoring monitoring \
  -o jsonpath='{.status.conditions[-1]}' \
  | python3 -c "import sys,json; c=json.load(sys.stdin); print(c['type'], c['status'], c.get('message','')[:120])"
```

---

## Flux

### Force reconcile any HelmRelease immediately

```bash
kubectl annotate helmrelease <name> -n <namespace> \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --overwrite
```

### Suspend / resume a HelmRelease

Suspend stops Flux from reconciling the release. Use this before a manual migration to prevent Flux from fighting your changes.

```bash
# Suspend
kubectl patch helmrelease <name> -n <namespace> \
  --type=merge -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch helmrelease <name> -n <namespace> \
  --type=merge -p '{"spec":{"suspend":false}}'
```

### Check what all HelmReleases are doing

```bash
kubectl get helmrelease -A
```

### Check why a HelmRelease is failing

```bash
kubectl describe helmrelease <name> -n <namespace> | grep -A10 "Status:\|Message:\|Reason:"
```

### Flux prune is off — manual resource cleanup required

The apps Kustomization has `prune: false`. Removing a file from Git does **not** delete the cluster resource. You must delete it manually:

```bash
kubectl delete <kind> -n <namespace> <name>
```

---

## CNPG

### Check cluster health

```bash
kubectl get cluster -A
# Look for STATUS = "Cluster in healthy state" and READY matching INSTANCES
```

### Get the app user connection details

CNPG auto-generates credentials into `<cluster-name>-app`:

```bash
kubectl get secret -n <namespace> <cluster-name>-app \
  -o jsonpath='{.data}' \
  | python3 -c "import sys,json,base64; d=json.load(sys.stdin); [print(k,'=',base64.b64decode(v).decode()) for k,v in d.items()]"
```

### Connect to a CNPG cluster interactively

```bash
CNPG_PASS=$(kubectl get secret -n <namespace> <cluster-name>-app \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl exec -it -n <namespace> <cluster-name>-1 -- \
  sh -c "PGPASSWORD='${CNPG_PASS}' psql -U <dbuser> -h <cluster-name>-rw <dbname>"
```

For Authentik specifically:
```bash
CNPG_PASS=$(kubectl get secret -n authentik authentik-cnpg-app \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl exec -it -n authentik authentik-cnpg-1 -- \
  sh -c "PGPASSWORD='${CNPG_PASS}' psql -U authentik -h authentik-cnpg-rw authentik"
```

### Check backup status

```bash
kubectl get backup -n <namespace>
kubectl get scheduledbackup -n <namespace>
```

### Dump a CNPG database to stdout

```bash
CNPG_PASS=$(kubectl get secret -n <namespace> <cluster-name>-app \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl exec -n <namespace> <cluster-name>-1 -- \
  sh -c "PGPASSWORD='${CNPG_PASS}' pg_dump -U <dbuser> -h <cluster-name>-rw <dbname>" \
  > dump.sql
```

---

## Sealed Secrets

### Re-seal a secret (after adding/removing keys)

Requires the `kubeseal` binary. If not installed:

```bash
# Check controller version first
kubectl get pod -n kube-system -l app.kubernetes.io/name=sealed-secrets \
  -o jsonpath='{.items[0].spec.containers[0].image}'

# Download matching kubeseal (adjust version)
curl -sL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.28.0/kubeseal-0.28.0-linux-amd64.tar.gz" \
  | tar xz -C /tmp kubeseal
```

Fetch the controller's public cert:
```bash
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o jsonpath='{.items[0].data.tls\.crt}' | base64 -d > /tmp/sealed-secrets.crt
```

Seal a new or updated secret:
```bash
kubectl create secret generic <name> \
  --namespace <namespace> \
  --from-literal=KEY1="value1" \
  --from-literal=KEY2="value2" \
  --dry-run=client -o yaml \
| /tmp/kubeseal \
    --cert /tmp/sealed-secrets.crt \
    --format yaml > path/to/sealed-secret.yaml
```

### Read a live (decrypted) secret value

```bash
kubectl get secret -n <namespace> <name> \
  -o jsonpath='{.data.<key>}' | base64 -d
```

---

## General Pod Ops

### Restart a deployment cleanly

```bash
kubectl rollout restart deployment -n <namespace> <name>
kubectl rollout status deployment -n <namespace> <name> --timeout=120s
```

### Delete a specific pod immediately (deployment will recreate it)

```bash
kubectl delete pod -n <namespace> <pod-name>
```

### Delete all pods matching a label (deployment recreates them)

```bash
kubectl delete pod -n <namespace> -l <label-key>=<label-value>
```

### Wait for a resource condition

```bash
# Wait for a pod to be ready
kubectl wait pod -n <namespace> -l <label> --for=condition=Ready --timeout=120s

# Wait for a CNPG cluster to be healthy
kubectl wait cluster -n <namespace> <name> --for=condition=Ready --timeout=180s

# Wait for a HelmRelease to finish
kubectl wait helmrelease -n <namespace> <name> --for=condition=Ready --timeout=300s
```

### Tail logs across all pods matching a label

```bash
kubectl logs -n <namespace> -l <label-key>=<label-value> --all-containers --follow
```

### Check events for a namespace (sorted by time)

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```

### Check resource usage across nodes

```bash
kubectl top nodes
kubectl top pods -n <namespace>
```
