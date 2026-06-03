# Cluster Maintenance Runbook

**Cluster:** safeqbit-local-hq  
**Last updated:** 2026-05-29 (added Graceful Node Shutdown section)

A living document. Add entries as new patterns emerge.

**Related docs:**
- [backup-strategy.md](backup-strategy.md) - full backup schedule + retention reference
- [improvement-plan.md](improvement-plan.md) - prioritized tech-debt backlog
- [cnpg-strategy.md](cnpg-strategy.md) - per-cluster CNPG sizing rationale

---

## Table of Contents

- [Node Shutdown (Hardware Maintenance)](#node-shutdown-hardware-maintenance)
- [Graceful Node Shutdown (kubelet)](#graceful-node-shutdown-kubelet)
- [Flannel VXLAN Checksum Offload (Post-Reboot)](#flannel-vxlan-checksum-offload-post-reboot)
- [Longhorn Orphan Replica Cleanup](#longhorn-orphan-replica-cleanup)
- [CoreDNS HA](#coredns-ha)
- [Grafana](#grafana)
- [Flux](#flux)
- [CNPG](#cnpg)
- [Sealed Secrets](#sealed-secrets)
- [Monitoring & Alerting](#monitoring--alerting)
- [General Pod Ops](#general-pod-ops)

---

## Node Shutdown (Hardware Maintenance)

Use this procedure when taking a node offline for hardware changes - RAM, disk, NIC, BIOS update, etc.  
**Estimated downtime per app:** 1-5 min for stateless pods, longer if Longhorn replicas must rebuild.

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

Do not proceed if any CNPG cluster is not ready or any Longhorn volume is `degraded` - fix those first or you risk data loss.

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
# Patch replica count to 2 (from 1) - Longhorn will schedule a replica on another node
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

- `--ignore-daemonsets`: DaemonSet pods (Longhorn engine, log collectors) are left - this is correct.
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

# CNPG clusters should still be healthy (may take 1-2 min to elect new primary if applicable)
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

After the node comes back and is uncordoned, CNPG will reschedule on any available node - it does not pin to the original node.

#### All three nodes are etcd members - drain ONE at a time

This cluster runs 3 k3s servers, all `control-plane,etcd` (no agent-only nodes). etcd needs a quorum of 2 of 3 to stay writable, so this is the single hardest constraint on node maintenance:

- **Never take down more than one node at a time** for planned maintenance. With one server off, etcd still has 2/3 quorum and the kube-apiserver on the surviving two keeps `kubectl` working. Take a *second* server down before the first returns and you lose quorum → the API goes unavailable until a node rejoins.
- Before shutting down the *next* node, confirm the previous one has fully rejoined:
  ```bash
  kubectl get nodes                          # all 3 Ready
  kubectl get --raw='/healthz/etcd' ; echo   # returns: ok
  ```
- Complete all drain/cordon steps _before_ powering the node off.

(For reference: a single-server k3s cluster has no quorum protection - the entire API is down whenever the server is off. Not applicable here, but noted for context.)

---

## Graceful Node Shutdown (kubelet)

**What it is:** kubelet's built-in *Graceful Node Shutdown* (GA since k8s 1.21). When the OS begins a clean shutdown (systemd `poweroff`/`reboot`, Proxmox ACPI "Shutdown", a UPS-triggered halt), kubelet grabs a systemd inhibitor lock and terminates pods in order - regular pods first, then critical pods (CNI, Longhorn, kube-system) - respecting each pod's termination grace, *before* the OS powers off.

**Why we want it:** Without it (the prior state - `shutdownGracePeriod: 0s`), a clean shutdown that *skips* the full drain runbook above (a forgotten step, an ACPI shutdown from Proxmox, a power event) hard-kills every pod the instant kubelet dies with the OS. Stateful pods (CNPG postgres, Longhorn engines) get no chance to flush. This is the **safety floor**: it does **not** cordon, migrate Longhorn replicas, or protect etcd quorum - for planned hardware maintenance you still run the full [Node Shutdown](#node-shutdown-hardware-maintenance) runbook. It just stops the worst-case hard-kill when the runbook isn't used.

### The InhibitDelayMaxSec gotcha (mandatory companion setting)

systemd-logind's default inhibitor cap (`InhibitDelayMaxSec`) is **5 seconds**. kubelet's graceful shutdown holds a logind inhibitor lock; if `shutdownGracePeriod` exceeds `InhibitDelayMaxSec`, logind preempts it after 5s and the grace window is silently useless. So `InhibitDelayMaxSec` **must** be raised to ≥ `shutdownGracePeriod`. This is not optional - it's the difference between the feature working and not.

### Configuration (all 3 nodes)

Chosen values: 120s total, of which the last 30s is reserved for critical pods (so regular pods get 90s). `InhibitDelayMaxSec` set a touch above total so logind never preempts kubelet.

| Setting | Where | Value |
|---|---|---|
| `shutdown-grace-period` | `/etc/rancher/k3s/config.yaml` (kubelet-arg) | `120s` |
| `shutdown-grace-period-critical-pods` | same | `30s` |
| `InhibitDelayMaxSec` | `/etc/systemd/logind.conf.d/10-k3s-graceful-shutdown.conf` | `130` |

### Apply runbook - ONE node at a time (preserve etcd quorum)

Run on each node in turn; do **not** start the next node until the current one is back `Ready` and `/healthz/etcd` returns `ok` (see [etcd quorum rule](#all-three-nodes-are-etcd-members--drain-one-at-a-time)).

```bash
# --- on the target node (ssh in) ---

# 1. Add kubelet args. NOTE: if /etc/rancher/k3s/config.yaml already has a
#    `kubelet-arg:` key (e.g. once P1.2 etcd-s3 lands), MERGE into that list
#    by hand - do not append a second `kubelet-arg:` block (duplicate YAML key).
sudo test -f /etc/rancher/k3s/config.yaml && grep -q kubelet-arg /etc/rancher/k3s/config.yaml \
  && echo "EDIT BY HAND - kubelet-arg already present" \
  || sudo tee -a /etc/rancher/k3s/config.yaml >/dev/null <<'EOF'
kubelet-arg:
  - "shutdown-grace-period=120s"
  - "shutdown-grace-period-critical-pods=30s"
EOF

# 2. Raise the logind inhibitor cap (drop-in, leaves the stock logind.conf untouched)
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/10-k3s-graceful-shutdown.conf >/dev/null <<'EOF'
[Login]
InhibitDelayMaxSec=130
EOF
sudo systemctl restart systemd-logind          # safe - does not kill existing sessions

# 3. Apply the kubelet change by restarting k3s (brief apiserver blip on THIS node;
#    HA covers it. flannel-fix-offload.service re-runs automatically via BindsTo=k3s.service)
sudo systemctl restart k3s

# --- from any working kubectl ---

# 4. Verify the node came back and etcd is healthy BEFORE doing the next node
kubectl get nodes                               # all 3 Ready
kubectl get --raw='/healthz/etcd' ; echo        # ok

# 5. Confirm the setting actually took on this node
NODE=k3s-server-0X
kubectl get --raw "/api/v1/nodes/$NODE/proxy/configz" \
  | python3 -c "import sys,json;d=json.load(sys.stdin)['kubeletconfig'];print('grace',d.get('shutdownGracePeriod'),'critical',d.get('shutdownGracePeriodCriticalPods'))"
# Expect: grace 2m0s critical 30s
```

**Verify it actually fires** (optional, low-risk): on a maintenance window, `sudo systemctl reboot` one node and watch - pods should show a `Terminated` / node-shutdown status and the host should pause ~90s before going down, rather than dropping instantly.

**Status:** Config documented here; applied per-node via the runbook above. Re-check after any k3s upgrade or if `config.yaml` is regenerated.

> This covers Layer 0 of the node-shutdown plan. Layers 1 (orchestrated on-demand safe shutdown) and 2 (power-event / UPS-triggered) are tracked in [improvement-plan.md](improvement-plan.md) as P2.7 and P3.6.

---

## Flannel VXLAN Checksum Offload (Post-Reboot)

**Symptom seen on 2026-05-27 after rebooting k3s-server-02:**
- Pods on the rebooted node can ping pod IPs across nodes via ICMP (≈0.3 ms), but every TCP/UDP connection to a ClusterIP or remote pod IP times out.
- DNS from any pod on the affected node fails (`dig @10.43.0.10` times out).
- `kubectl exec ... -- nc -zv 10.43.0.1 443` succeeds (kubernetes API uses host-network backend, no flannel crossing); `nc -zv 10.43.134.108 9500` (longhorn-backend, pod-network backend) times out.
- Cluster-internal services backed by pods on **other** nodes are unreachable; services whose backend is local to the affected node, or backed by host-network pods, work fine.
- `longhorn-csi-plugin` and `longhorn-manager` CrashLoopBackOff on the affected node because they cannot reach `longhorn-backend` / `longhorn-admission-webhook`.
- `kubectl get events`: snapshot operations show `connect: connection refused` to `10.43.134.108:9500`.

**Root cause:** On Proxmox `virtio_net` VMs, the `flannel.1` VXLAN interface comes up with `tx-checksum-ip-generic: on`. This corrupts the inner-packet checksums of TCP/UDP carried over VXLAN. Receiving nodes silently drop the bad-checksum packets. ICMP is not affected because of how kernel computes ICMP checksums.

**Diagnosis:**
```bash
# Pick the affected node, e.g. k3s-server-02
kubectl debug node/k3s-server-02 --image=nicolaka/netshoot --profile=sysadmin -- sh -c '
  nsenter -t 1 -m -n -- ethtool -k flannel.1 | grep tx-checksum-ip-generic
  nsenter -t 1 -m -n -- ip -s link show flannel.1 | tail -5
'
# A broken node shows tx-checksum-ip-generic: on AND extremely asymmetric RX vs TX
# (e.g. RX 304 packets / TX 58k packets since the interface came up).
```

**Manual fix (volatile, reverts on next k3s restart / reboot):**
```bash
kubectl debug node/<node> --image=nicolaka/netshoot --profile=sysadmin -- \
  sh -c 'nsenter -t 1 -m -n -- ethtool -K flannel.1 tx-checksum-ip-generic off'
```

**Persistent fix (already installed on all 3 nodes 2026-05-27):**

`/etc/systemd/system/flannel-fix-offload.service`:
```ini
[Unit]
Description=Disable tx-checksum-ip-generic on flannel.1 (Proxmox virtio_net VXLAN bug)
After=k3s.service
BindsTo=k3s.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c 'for i in $(seq 1 120); do ip link show flannel.1 >/dev/null 2>&1 && /usr/sbin/ethtool -K flannel.1 tx-checksum-ip-generic off && exit 0; sleep 2; done; exit 1'

[Install]
WantedBy=multi-user.target
```

`BindsTo=k3s.service` makes the unit re-run whenever k3s restarts (which recreates `flannel.1`). Verify after any node reboot:
```bash
systemctl status flannel-fix-offload.service
ethtool -k flannel.1 | grep tx-checksum-ip-generic   # should be "off"
```

---

## Longhorn Orphan Replica Cleanup

**Symptom:** A Longhorn volume stays `degraded` even though all nodes appear ready. Volume conditions show `Scheduled: False, ReplicaSchedulingFailure: precheck new replica failed: insufficient storage`. Disk usage in Longhorn (`storageScheduled`) on the affected node is well below the disk size, but `df -h /var/lib/longhorn` shows it nearly full. Cross-referencing `kubectl -n longhorn-system get replicas.longhorn.io --field-selector spec.nodeID=<node>` against `ls /var/lib/longhorn/replicas/` reveals more on-disk directories than Longhorn knows about.

**Root cause:** When a replica fails or is rebuilt, the old directory is sometimes not deleted from disk - Longhorn drops the CR but the data sits there forever, consuming space the scheduler doesn't account for. These show up as `Orphan` CRs in `longhorn-system`. Hit on 2026-05-27 on `k3s-server-02`: 14 orphan dirs (~100 GiB) blocked the Prometheus PVC from gaining its 3rd replica.

**Diagnosis:**
```bash
# How many Longhorn-known replicas vs how many on-disk?
kubectl -n longhorn-system get replicas.longhorn.io --no-headers | awk '$4=="<node>"' | wc -l
# vs
kubectl debug node/<node> --image=busybox --profile=sysadmin -- \
  sh -c 'nsenter -t 1 -m -n -- ls /var/lib/longhorn/replicas/ | wc -l'

# Orphans Longhorn has already detected:
kubectl -n longhorn-system get orphans.longhorn.io
```

**Fix (cluster-wide, in repo):** `infrastructure/safeqbit-local-hq/controllers/longhorn.yaml` sets `defaultSettings.orphanResourceAutoDeletion: replica-data` and `orphanResourceAutoDeletionGracePeriod: 300`. With these set, Longhorn auto-deletes orphan replica data 5 min after detection.

**One-off fix (live cluster, if Helm value isn't applied yet):**
```bash
kubectl -n longhorn-system patch settings.longhorn.io orphan-resource-auto-deletion \
  --type=merge -p '{"value":"replica-data"}'
# Wait 5 min, then verify
kubectl -n longhorn-system get orphans.longhorn.io      # should be empty
kubectl -n longhorn-system get volumes.longhorn.io      # robustness=healthy after rebuild
```

After cleanup, Longhorn will auto-schedule the missing replica on the freed node and rebuild from existing healthy replicas (status will show `WO` mode during sync, then transition to `RW` when complete).

---

## CoreDNS HA

k3s deploys CoreDNS as an **Addon** (CR `addons.k3s.cattle.io/coredns`), not as a HelmChart, so `HelmChartConfig` overrides don't apply. The source manifest lives at `/var/lib/rancher/k3s/server/manifests/coredns.yaml` on every control-plane node and the Addon controller watches that file's checksum - editing the file makes k3s re-apply the new content.

**Current persistent config (set 2026-05-28):**
- `replicas: 3` in the Deployment spec (injected on line 88, before `revisionHistoryLimit: 0`)
- The file already ships with `topologySpreadConstraints` on `kubernetes.io/hostname` with `whenUnsatisfiable: DoNotSchedule`, which enforces one pod per host

**Verify the edit is in place on each control-plane node:**
```bash
for n in k3s-server-01 k3s-server-02 k3s-server-03; do
  kubectl debug node/$n --image=busybox --profile=sysadmin -- sh -c \
    'nsenter -t 1 -m -n -- grep -nE "^  replicas:" /var/lib/rancher/k3s/server/manifests/coredns.yaml'
done
# Each line should print:  88:  replicas: 3
```

**If a node is missing the edit** (e.g., after a fresh node rebuild or k3s reinstall):
```bash
kubectl debug node/<node> --image=busybox --profile=sysadmin -- sh -c '
  nsenter -t 1 -m -n -- sh -c "
    F=/var/lib/rancher/k3s/server/manifests/coredns.yaml
    grep -q \"^  replicas: 3$\" \$F && exit 0
    cp \$F \${F}.bak.\$(date +%s)
    sed -i \"/^  revisionHistoryLimit: 0$/i\\\\  replicas: 3\" \$F
  "'
```

**After a k3s version upgrade:** k3s may regenerate `coredns.yaml` from its embedded template, wiping the edit. Re-check with the verify-loop above; re-inject if needed.

**Long-term cleaner alternative (P3 in improvement-plan):** add `--disable=coredns` to k3s server config, deploy CoreDNS via Flux/Helm with our own values. Removes the need for node-local file edits entirely.

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

**Fix - restart the pod:**
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

### Flux prune is off - manual resource cleanup required

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

## Monitoring & Alerting

The cluster is monitored by kube-prometheus-stack in the `monitoring` namespace. Prometheus picks up `ServiceMonitor` / `PodMonitor` / `PrometheusRule` resources labeled `release: monitoring` from any namespace. Alertmanager routes to Slack via the `alertmanager-slack-webhook` SealedSecret.

### What's scraped

| Source | Mechanism | File |
|---|---|---|
| kubelet, apiserver, node-exporter, coredns, etcd | chart defaults | upstream |
| All CNPG clusters (per-pod postgres metrics) | PodMonitor auto-created by CNPG operator | `cnpg-system`/per-ns |
| Longhorn manager (`longhorn-backend.longhorn-system:9500`) | ServiceMonitor `longhorn-manager` | `configs/monitoring-servicemonitors.yaml` |
| Velero (`velero.velero:8085`) | ServiceMonitor `velero` | `configs/monitoring-servicemonitors.yaml` |
| Cloudflared tunnels | per-app ServiceMonitor | `apps/.../cloudflared/` |

### Custom alert rules

`configs/monitoring-alert-rules.yaml` (PrometheusRule `safeqbit-storage-and-backup`) covers gaps in the upstream chart:

- **Longhorn**: `LonghornVolumeDegraded`, `LonghornVolumeFaulted`, `LonghornVolumeSnapshotCountHigh` (>200, prevent hitting the 250 cap), `LonghornNodeStorageLow` (>85%)
- **CNPG**: `CNPGReplicationLagHigh` (>5m lag), `CNPGExporterDown`
- **Velero**: `VeleroBackupFailed` (last status = 0), `VeleroBackupFailureRateHigh` (>2 in 24h)

### Suppressed false-positives

Three default kube-prometheus alerts are hard-disabled (not just silenced) because they target k8s control-plane components that k3s embeds into the server binary, so the standalone metric endpoints don't exist:

- `KubeProxyDown`, `KubeSchedulerDown`, `KubeControllerManagerDown`

These are off via `controllers/monitoring.yaml`:
```yaml
defaultRules:
  rules:
    kubeProxy: false
    kubeScheduler: false
    kubeControllerManager: false
kubeProxy:
  enabled: false
# ...same for the other two
```

### Verifying alerts

```bash
# Check Prometheus is scraping each target
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090
# Browse to http://localhost:9090/targets - all should be UP

# List active alerts
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-alertmanager 9093
# Browse to http://localhost:9093/#/alerts

# Trigger a known test alert (scale a deploy to 0)
kubectl -n photoprism scale deploy photoprism --replicas=0
# Wait 10m → KubePodNotReady/KubeDeploymentReplicasMismatch should fire to Slack
kubectl -n photoprism scale deploy photoprism --replicas=1
```

### Common alert investigation

| Alert | First thing to check |
|---|---|
| `LonghornVolumeDegraded` | `kubectl -n longhorn-system get volumes <vol> -o yaml` - which replica is missing? Often resolves on its own after node recovery. |
| `LonghornVolumeSnapshotCountHigh` | `kubectl get backups.postgresql.cnpg.io -A` - old CNPG Backup CRs likely accumulating. Verify `cnpg-backup-retention` CronJob is running daily. |
| `VeleroBackupFailed` | `velero backup describe <name> --details`; check `kubectl -n velero get backuprepository <ns>-<id>` for kopia maintenance errors. |
| `CNPGReplicationLagHigh` | `kubectl -n <ns> exec -ti <primary> -- psql -c "SELECT * FROM pg_stat_replication;"` |

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
