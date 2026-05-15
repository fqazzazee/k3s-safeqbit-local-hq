# Cluster Maintenance Runbook

**Cluster:** safeqbit-local-hq  
**Last updated:** 2026-05-15 (node-shutdown procedure added)

A living document. Add entries as new patterns emerge.

---

## Table of Contents

- [Node Shutdown (Hardware Maintenance)](#node-shutdown-hardware-maintenance)
- [Grafana](#grafana)
- [Flux](#flux)
- [CNPG](#cnpg)
- [Sealed Secrets](#sealed-secrets)
- [General Pod Ops](#general-pod-ops)

---

## Node Shutdown (Hardware Maintenance)

Use this procedure when taking a node offline for hardware changes — RAM, disk, NIC, BIOS update, etc.  
**Estimated downtime per app:** 1–5 min for stateless pods, longer if Longhorn replicas must rebuild.

### 1. Pre-flight checks

Verify the cluster is healthy before touching anything:

```bash
# All nodes ready
kubectl get nodes

# All CNPG clusters healthy
kubectl get cluster -A
# Expected: STATUS = "Cluster in healthy state", READY = INSTANCES for all rows

# All Longhorn volumes healthy (no degraded replicas)
kubectl get volumes.longhorn.io -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness'
# Expected: state=attached or detached, robustness=healthy for all

# Overall pod health
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded \
  | grep -v Completed
# Should return nothing (or only expected pending/init pods)
```

Do not proceed if any CNPG cluster is not ready or any Longhorn volume is `degraded` — fix those first or you risk data loss.

---

### 2. Handle Longhorn before draining

Longhorn volumes with only one replica **on the target node** will block a drain and leave pods stuck. Check and fix replication first.

**Identify under-replicated volumes on the target node:**
```bash
NODE=<node-name>   # e.g. k3s-worker-01

kubectl get replicas.longhorn.io -n longhorn-system \
  -o custom-columns='VOLUME:.spec.volumeName,NODE:.spec.nodeID,STATE:.status.currentState' \
  | grep "$NODE"
```

For any volume that has its **only** replica on the target node, increase the replica count temporarily so Longhorn places a copy elsewhere:

```bash
# Patch replica count to 2 (from 1) — Longhorn will schedule a replica on another node
kubectl patch volume.longhorn.io <volume-name> -n longhorn-system \
  --type=merge -p '{"spec":{"numberOfReplicas":2}}'

# Wait until the new replica is healthy before proceeding
kubectl get replicas.longhorn.io -n longhorn-system \
  -o custom-columns='VOLUME:.spec.volumeName,NODE:.spec.nodeID,STATE:.status.currentState' \
  | grep <volume-name>
# Wait for all replicas to show state=running
```

Repeat for each under-replicated volume. Only drain once all volumes show `robustness=healthy`.

**Enable Longhorn node drain policy** (tells Longhorn to cooperate with the drain):
```bash
kubectl annotate node "$NODE" \
  node.longhorn.io/default-disks-config='[{"allowScheduling":false}]' \
  --overwrite
```

Or use the Longhorn UI: **Node** → select the node → **Edit** → toggle **Scheduling: Disabled**.

---

### 3. Suspend Flux (prevent reconcile fights during drain)

```bash
# Suspend all kustomizations so Flux doesn't fight pod evictions
kubectl patch kustomization apps -n flux-system \
  --type=merge -p '{"spec":{"suspend":true}}'
kubectl patch kustomization infrastructure -n flux-system \
  --type=merge -p '{"spec":{"suspend":true}}'

# Confirm
kubectl get kustomization -n flux-system
```

---

### 4. Cordon and drain the node

**Cordon** marks the node unschedulable (no new pods land on it):
```bash
NODE=<node-name>
kubectl cordon "$NODE"
kubectl get node "$NODE"   # Should show SchedulingDisabled
```

**Drain** evicts all running pods (except DaemonSets, which stay until node is gone):
```bash
kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s
```

- `--ignore-daemonsets`: DaemonSet pods (Longhorn engine, log collectors) are left — this is correct.
- `--delete-emptydir-data`: Required for any pod using `emptyDir` volumes (e.g. AFFiNE Redis).
- `--grace-period=60`: Gives each pod 60 s to flush/shutdown cleanly.
- `--timeout=300s`: Abort if drain takes more than 5 min (investigate, don't leave hanging).

**If drain is stuck** (common cause: a pod's PodDisruptionBudget blocks eviction):
```bash
# See which pods haven't evicted yet
kubectl get pods -A --field-selector=spec.nodeName="$NODE"

# Check for PDBs blocking eviction
kubectl get pdb -A

# Force eviction with a longer timeout or delete the blocking pod manually
kubectl delete pod -n <namespace> <stuck-pod-name> --grace-period=30
```

---

### 5. Verify workloads rescheduled

After drain completes, all non-DaemonSet pods should have moved to other nodes:

```bash
# No non-DaemonSet pods should remain on the node
kubectl get pods -A --field-selector=spec.nodeName="$NODE"

# Check that critical workloads came back up elsewhere
kubectl get pods -A | grep -E "0/[0-9]|Pending|CrashLoop|Error"

# CNPG clusters should still be healthy (may take 1–2 min to elect new primary if applicable)
kubectl get cluster -A
```

---

### 6. Perform the hardware work

The node is now safe to power off:

```bash
# SSH to the node and shut it down gracefully
ssh <user>@<node-ip> "sudo shutdown -h now"
```

Do the hardware change. Power the node back on when done.

---

### 7. Bring the node back online

Wait for the node to rejoin the cluster (watch with `-w`):

```bash
kubectl get nodes -w
# Wait until <node-name> transitions to Ready
```

**Uncordon** to allow pods to schedule on it again:
```bash
kubectl uncordon "$NODE"
```

**Re-enable Longhorn disk scheduling on the node:**
```bash
# Remove the annotation set in step 2
kubectl annotate node "$NODE" node.longhorn.io/default-disks-config- --overwrite
```
Or via Longhorn UI: **Node** → **Edit** → toggle **Scheduling: Enabled**.

**Resume Flux:**
```bash
kubectl patch kustomization apps -n flux-system \
  --type=merge -p '{"spec":{"suspend":false}}'
kubectl patch kustomization infrastructure -n flux-system \
  --type=merge -p '{"spec":{"suspend":false}}'
```

**Check Longhorn volume rebalancing:** Longhorn won't automatically move replicas back, but the node is now available for future replica placement. If you increased any volume's replica count in step 2, restore it:
```bash
kubectl patch volume.longhorn.io <volume-name> -n longhorn-system \
  --type=merge -p '{"spec":{"numberOfReplicas":1}}'
```

---

### 8. Post-maintenance health check

```bash
# All nodes ready
kubectl get nodes

# All pods healthy
kubectl get pods -A | grep -E "0/[0-9]|Pending|CrashLoop|Error"

# All CNPG clusters healthy
kubectl get cluster -A

# All Longhorn volumes healthy
kubectl get volumes.longhorn.io -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness'

# All HelmReleases reconciled
kubectl get helmrelease -A
```

---

### Special cases

#### CNPG single-instance clusters (instances: 1)

Most CNPG clusters in this repo use `instances: 1`. When the node hosting the postgres pod is drained, the pod evicts and must reschedule on another node. The Longhorn PVC will follow only if the volume already has a replica on the destination node (see step 2). **If the PVC only has one replica on the drained node, the pod will stay `Pending` until the node comes back.**

Mitigation: always run step 2 to ensure each CNPG PVC has at least one replica on a node that will survive the drain.

After the node comes back and is uncordoned, CNPG will reschedule on any available node — it does not pin to the original node.

#### Server (control plane) node vs. agent node

If the node being shut down is a **k3s server** (control plane), the kube-apiserver and embedded etcd will be unavailable during the outage. `kubectl` will stop working from the moment the node goes down until it rejoins. Plan accordingly: complete all drain/cordon steps _before_ shutting the server node down, and accept that the cluster API is unreachable during the maintenance window.

For a **single-server k3s cluster** (no HA), this means the entire cluster API is down while the server node is off. Agent nodes continue running their existing pods but no new scheduling happens.

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
