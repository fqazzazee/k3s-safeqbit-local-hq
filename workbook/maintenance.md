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
- [etcd Snapshots (local + B2)](#etcd-snapshots-local--b2)
- [Flannel VXLAN Checksum Offload (Post-Reboot)](#flannel-vxlan-checksum-offload-post-reboot)
- [Longhorn Orphan Replica Cleanup](#longhorn-orphan-replica-cleanup)
- [Longhorn Capacity & Disk Expansion](#longhorn-capacity--disk-expansion)
- [CoreDNS HA](#coredns-ha)
- [Grafana](#grafana)
- [Prometheus TSDB / Storage](#prometheus-tsdb--storage)
- [Flux](#flux)
- [CNPG](#cnpg)
- [Velero](#velero)
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

## etcd Snapshots (local + B2)

k3s snapshots etcd **weekly (Sunday 00:00 UTC)** on every server node and
uploads each snapshot to B2 (`k3s-safeqbit-etcd` bucket, `snapshots/`
folder) via the `etcd-s3` block in `/etc/rancher/k3s/config.yaml`
(node-local, contains the bucket-scoped B2 key — never in Git). Retention 8
per node (~8 weeks); bucket lifecycle keeps deleted versions 30 days as an
undelete window. Design rationale and restore procedure:
[backup-strategy.md → Layer 0](backup-strategy.md#layer-0---etcd-snapshots-to-b2-control-plane);
node config reference: [node-bootstrap.md](node-bootstrap.md).

```bash
# Verify uploads are happening (records appear after each scheduled run)
kubectl get etcdsnapshotfiles | grep 's3://'

# On-demand snapshot + upload (run on any server node)
sudo k3s etcd-snapshot save
# ("Unknown flag --tls-san" warning from this subcommand is harmless)

# List what's on the nodes
kubectl get etcdsnapshotfiles | grep 'file://'
```

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

## Longhorn Capacity & Disk Expansion

Longhorn enforces **two independent limits** that fail differently. The orphan case
above is the *physical* kind; this section is the *scheduling* kind, plus how to add
real capacity and how to recover a stuck PVC expansion.

- **Physical** — actual bytes on `/var/lib/longhorn` (the `sdb` disk). Alert: `LonghornNodeStorageLow` (`storage_usage / storage_capacity > 0.85`).
- **Scheduling (over-provisioning)** — sum of *nominal* replica sizes vs the limit `(storageMaximum − storageReserved) × storageOverProvisioningPercentage/100`. With `storageOverProvisioningPercentage: 100` (kept strict in `controllers/longhorn.yaml`) the limit ≈ disk size. Alert: `LonghornNodeSchedulingCeiling`.

### Symptom: ReplicaSchedulingFailure while the disk is NOT physically full

`kubectl -n longhorn-system get volumes <vol> -o yaml` shows `Scheduled: False
(ReplicaSchedulingFailure)`, but `df -h /var/lib/longhorn` has free space. New or
expanded PVCs, **and every Velero data-mover backup** (their temp restore volumes
can't place), fail. Hit 2026-06-28 after a Prometheus PVC expansion ate the last
over-provisioning headroom (441 MiB/node).

**Check scheduling headroom per node:**
```bash
kubectl get nodes.longhorn.io -n longhorn-system -o jsonpath='{range .items[*]}{.metadata.name}: scheduled={.status.diskStatus.*.storageScheduled} max={.status.diskStatus.*.storageMaximum} reserved={.status.diskStatus.*.storageReserved}{"\n"}{end}'
# Headroom = (max - reserved) - scheduled  (at over-provisioning 100%)
```

**Relieve it (in order of preference):**
1. **Reclaim** — delete unused PVCs / orphaned namespaces (frees `scheduled` bytes). Deleting a namespace cascades to its PVCs → Longhorn volumes. (2026-06-28: removed the orphaned `sra-dev-demo` namespace, +5 Gi/node.)
2. **Grow the sdb disk** (below) — the durable fix.
3. **Raise `storageOverProvisioningPercentage`** in `controllers/longhorn.yaml` — quick, but overcommits thin volumes; only safe when physical usage is low, and needs disk alerting. Kept at 100 here on purpose.

### Growing the Longhorn disk (sdb)

`/var/lib/longhorn` is a **bare XFS filesystem on `/dev/sdb` (no partition table)**.
After enlarging the virtual disk in Proxmox, run **on each host** (one at a time):
```bash
# 1. Confirm the kernel sees the new size (usually auto-detected for sdb):
lsblk /dev/sdb
#    If it still shows the old size, rescan:
echo 1 | sudo tee /sys/block/sdb/device/rescan

# 2. Grow the XFS filesystem ONLINE. xfs_growfs takes the MOUNTPOINT, not the device.
#    It is XFS, not ext4 — do NOT use resize2fs. No growpart needed (bare disk).
sudo xfs_growfs /var/lib/longhorn

# 3. Verify:
df -h /var/lib/longhorn
```
Longhorn polls capacity and updates `storageMaximum` within ~1 min — no restart.
Confirm cluster-side:
```bash
kubectl get nodes.longhorn.io -n longhorn-system -o jsonpath='{range .items[*]}{.metadata.name} max={.status.diskStatus.*.storageMaximum}{"\n"}{end}'
```

### Online PVC expansion (when you DO want a single volume bigger)

`longhorn` StorageClass has `allowVolumeExpansion: true`. Patch the PVC:
```bash
kubectl patch pvc -n <ns> <pvc> --type merge -p '{"spec":{"resources":{"requests":{"storage":"<new>Gi"}}}}'
```
Longhorn online-expands the replicas, then the filesystem resizes on the node.
If the admission webhook rejects it (`cannot schedule ... more bytes`), you are at
the scheduling ceiling — see above. **Note:** a StatefulSet's `volumeClaimTemplate`
is immutable, so the live PVC size can legitimately differ from the manifest
(e.g. the Prometheus PVC is `longhorn` while `controllers/monitoring.yaml` still
says `nfs-truenas`/30Gi — the manifest only applies on a fresh redeploy).

### Stuck PVC expansion — stale iscsid PID / iSCSI frontend refresh

**Symptom:** after patching a PVC larger, Longhorn expands the *replicas* but the
volume stays at the old size with `expansionRequired: true` forever. The
instance-manager on the volume's node logs (every few seconds):
```
fail to refresh iSCSI initiator ... nsenter: cannot open /host/proc/<PID>/ns/mnt: No such file or directory
```
**Root cause:** the node's `iscsid` restarted (e.g. during an OS upgrade) and
Longhorn cached the dead PID, so the online frontend rescan never completes — and
the volume won't detach to do an offline resize because the volume-expansion-
controller keeps it attached. Hit 2026-06-28 on the Prometheus volume after Ubuntu
node upgrades.

**Diagnose:**
```bash
VOL=<pvc-...>   # Longhorn volume name (kubectl get pvc -n <ns> <pvc> -o jsonpath='{.spec.volumeName}')
kubectl get volumes.longhorn.io -n longhorn-system $VOL -o jsonpath='expReq={.status.expansionRequired} state={.status.state}{"\n"}'
IM=$(kubectl get pods -n longhorn-system -o wide | awk '/instance-manager/ && /<node>/{print $1; exit}')
kubectl logs -n longhorn-system "$IM" --since=5m | grep -i "fail to refresh iSCSI"
```

**Fix — scoped to one volume, NO data loss (replicas already hold the new size):**
```bash
# 1. Quiesce the workload so the volume has no consumer. For operator-managed
#    StatefulSets (Prometheus), scale the operator down too or it reconciles back.
kubectl scale deploy -n monitoring monitoring-kube-prometheus-operator --replicas=0
kubectl scale statefulset -n monitoring prometheus-monitoring-kube-prometheus-prometheus --replicas=0

# 2. Delete the volume's Engine CR. Longhorn recreates the engine; the fresh engine
#    binds to the LIVE iscsid and the pending (offline) expansion completes.
kubectl delete engines.longhorn.io -n longhorn-system ${VOL}-e-0

# 3. Wait for the expansion to finish:
until [ "$(kubectl get volumes.longhorn.io -n longhorn-system $VOL -o jsonpath='{.status.expansionRequired}')" = "false" ]; do sleep 5; done

# 4. Bring the workload back; the filesystem resizes online on reattach.
kubectl scale statefulset -n monitoring prometheus-monitoring-kube-prometheus-prometheus --replicas=1
kubectl scale deploy -n monitoring monitoring-kube-prometheus-operator --replicas=1
```
**Heavier node-wide alternative:** restarting the node's `instance-manager` pod
re-discovers iscsid — but it bounces **every** engine on that node (brief I/O pause
for all those volumes). Avoid unless many volumes on the node are affected.

---

## CoreDNS HA

> **Superseded 2026-07-03 (PR #27).** The 2026-05-28 approach documented here
> previously — editing `/var/lib/rancher/k3s/server/manifests/coredns.yaml`
> on each node — **did not survive**: the k3s Addon controller's applied
> state includes `replicas: 1`, and the late-June node reboots silently
> reverted both the live patch and the file edits, leaving DNS on a single
> pod while its PDB alerted. Do not use the file-edit approach.

**Current mechanism:** a Flux-managed **cluster-proportional-autoscaler**
(`infrastructure/safeqbit-local-hq/configs/coredns-autoscaler.yaml`,
image registry.k8s.io/cpa/cluster-proportional-autoscaler) targets
`deployment/coredns` with linear params `nodesPerReplica: 1, min: 2, max: 3`
→ 3 replicas, re-enforced every ~10s. When k3s re-applies its bundled
manifest (any server restart or upgrade) and resets replicas to 1, the CPA
corrects it within seconds — validated live during the 2026-07-03 rolling
k3s restarts (three consecutive Addon re-applies, DNS stayed 3/3).

Host spread needs no configuration: the k3s-bundled template already carries
hostname `topologySpreadConstraints` (`DoNotSchedule`) + required
podAntiAffinity, so the 3 replicas always land one per node.

**Verify:**
```bash
kubectl -n kube-system get deploy coredns              # 3/3
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide   # one per node
kubectl -n kube-system logs deploy/coredns-autoscaler --tail=5
```

**If CoreDNS is at 1 replica:** check the autoscaler pod first — if it's
gone, Flux (kustomization `infrastructure-configs`) should recreate it; the
scale fixes itself once the CPA runs. Never patch the coredns Deployment
directly; the fix won't stick and the CPA makes it unnecessary.

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

## Prometheus TSDB / Storage

### Symptom: Grafana dashboards render but show "No data"

Panels load (Grafana itself is healthy) but every query is empty. The usual cause
is the **Prometheus TSDB volume is 100% full** — at 100% Prometheus can neither
append the WAL nor compact, so it stops ingesting and `up` returns nothing.
Hit 2026-06-28.

**Confirm:**
```bash
# Disk full?
kubectl exec -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -c prometheus -- df -h /prometheus
# Smoking gun in the logs:
kubectl logs -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -c prometheus --tail=20 | grep -i "no space"
#   ... write /prometheus/wal/...: no space left on device
# Ingestion check — an empty result means it is NOT ingesting:
kubectl exec -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -c prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query=up'
```

**Root cause:** the spec had `retention: 30d` (time-based) but **no `retentionSize`**
cap, so data outgrew the 30Gi volume before 30 days elapsed.

**Permanent fix (in repo):** `controllers/monitoring.yaml` →
`prometheusSpec.retentionSize: 33GB` (keep it comfortably below the volume size to
leave room for the WAL + compaction). Whichever of `retention` / `retentionSize` is
hit first triggers block deletion. Verify the live flags:
```bash
kubectl exec -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -c prometheus -- cat /proc/1/cmdline | tr '\0' '\n' | grep retention
# --storage.tsdb.retention.time=30d
# --storage.tsdb.retention.size=33GB
```

**Recovery when already wedged at 100%:** Prometheus can't self-trim in this state.
Free space first by **expanding the PVC** (see *Longhorn Capacity & Disk Expansion* —
and beware the stale-iscsid expansion deadlock, which bit this exact volume), then
the `retentionSize` cap keeps it bounded going forward. There is also a
`LonghornNodeStorageLow` / PVC-fill alert, but the real guard here is the size cap.

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

### Backup retention CronJob (prevents the 250-snapshot cap)

CNPG's `ScheduledBackup` has no built-in retention, so `cnpg-backup-retention`
(`configs/cnpg-backup-retention.yaml`, `cnpg-system`, 05:00 UTC daily) prunes old
Backup CRs (cascade-deletes their VolumeSnapshots → Longhorn snapshots). Keeps:
authentik 10, affine/netbox/monitoring 30.

**Gotcha (2026-06-28):** it was `ImagePullBackOff` because `bitnami/kubectl:1.34`
was removed from Docker Hub (Bitnami pulled all `docker.io/bitnami/*` tags, Aug 2025).
While it was dead ~30d, Backup CRs piled up (authentik hit 176). Now pinned to
`alpine/k8s:1.34.1` — it has kubectl **and a real `/bin/sh` + GNU coreutils**; the
prune script needs `head -n -N` (negative count), which busybox/distroless kubectl
images (`rancher/kubectl`, `registry.k8s.io/kubectl`) do **not** support.

```bash
# Health + manual run (e.g. after a long outage left a big backlog):
kubectl get cronjob -n cnpg-system cnpg-backup-retention
kubectl create job -n cnpg-system cnpg-retention-manual-$(date +%s) --from=cronjob/cnpg-backup-retention
kubectl logs -n cnpg-system -l job-name=cnpg-retention-manual-<...> -f
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

## Velero

Backups run as `Schedule` CRs in the `velero` namespace, store to Backblaze B2 (BSL
`default`), and move PVC data with the **kopia data mover** (`snapshotMoveData: true`).
Each namespace gets a `BackupRepository` named `<ns>-default-kopia`.

### Symptom: backups PartiallyFailed / "repository not initialized"

DataUploads fail with `error to connect to repository: repository not initialized
in the provided storage`. The `BackupRepository` CR can be stuck `Ready` while the
kopia repo in B2 is gone, so Velero never re-initializes it. The backup reports
**PartiallyFailed** (volume data NOT captured). Critically, `velero_backup_last_status`
reads **1 (success) even for PartiallyFailed**, which is why this hid for ~6 weeks on
`monitoring-bimonthly` (2026-06-28) — see the `VeleroBackupPartiallyFailed` alert added
since.

**Diagnose:**
```bash
# The repo whose lastMaintenanceTime lags far behind the others is the broken one:
kubectl get backuprepository.velero.io -n velero \
  -o custom-columns=NAME:.metadata.name,NS:.spec.volumeNamespace,PHASE:.status.phase,LASTMAINT:.status.lastMaintenanceTime
# Per-PVC failure detail for a backup:
kubectl get datauploads.velero.io -n velero -l velero.io/backup-name=<backup> \
  -o jsonpath='{range .items[*]}{.spec.sourcePVC} -> {.status.phase}: {.status.message}{"\n"}{end}'
```

**Fix — force re-initialization (deletes NO backup data in B2):**
```bash
kubectl delete backuprepository.velero.io -n velero <ns>-default-kopia
# Velero recreates + initializes it on the next backup that needs it.
```

**Validate cheaply** without moving a large PVC — back up one small PVC via a label
selector:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata: {name: <ns>-repo-check, namespace: velero}
spec:
  ttl: 24h0m0s
  storageLocation: default
  includedNamespaces: [<ns>]
  labelSelector: {matchLabels: {app.kubernetes.io/name: <small-app>}}   # e.g. alertmanager (1Gi)
  snapshotMoveData: true
  defaultVolumesToFsBackup: false
EOF
# Expect: DataUpload -> Completed, backup -> Completed, repo phase Ready w/ fresh lastMaintenanceTime.
```
**Gotcha:** a data-mover backup creates a **temp restore volume sized to the source
PVC**. If Longhorn is at the scheduling ceiling, that temp volume fails with
`ReplicaSchedulingFailure` and the backup hangs in `WaitingForPluginOperations`. See
*Longhorn Capacity & Disk Expansion*.

### Orphaned schedules

`infrastructure-configs` has `prune: false`, so deleting a `velero-schedule-*.yaml`
from git does **not** remove the live `Schedule`. After merge, delete it by hand:
```bash
kubectl delete schedule -n velero <name>   # e.g. pangolin-bimonthly, after the app was removed
```
(Metrics from removed schedules linger in Prometheus until the velero pod restarts —
cosmetic.)

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

- **Longhorn**: `LonghornVolumeDegraded`, `LonghornVolumeFaulted`, `LonghornVolumeSnapshotCountHigh` (>200, prevent hitting the 250 cap), `LonghornNodeStorageLow` (physical >85%), `LonghornNodeSchedulingCeiling` (scheduled/(capacity−reservation) >85% — the *scheduling* dimension `LonghornNodeStorageLow` misses; added 2026-06-28)
- **CNPG**: `CNPGReplicationLagHigh` (>5m lag), `CNPGExporterDown`
- **Velero**: `VeleroBackupFailed` (last status = 0 — full failures only), `VeleroBackupFailureRateHigh` (>2 in 24h), `VeleroBackupPartiallyFailed` (partial-failure counter over 16d, self-healing — catches the silent "data not captured" case `last_status` misses; added 2026-06-28)

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
| `LonghornVolumeSnapshotCountHigh` | `kubectl get backups.postgresql.cnpg.io -A` - old CNPG Backup CRs likely accumulating. Verify `cnpg-backup-retention` CronJob is running daily (see *CNPG → Backup retention CronJob*). |
| `LonghornNodeSchedulingCeiling` | Over-provisioning limit nearly reached. See *Longhorn Capacity & Disk Expansion* — reclaim volumes or grow the sdb disk; blocks new volumes + Velero data-mover temp volumes. |
| `VeleroBackupFailed` | `velero backup describe <name> --details`; check `kubectl -n velero get backuprepository <ns>-<id>` for kopia maintenance errors. |
| `VeleroBackupPartiallyFailed` | A backup ran but didn't capture volume data, and hasn't fully succeeded since. See *Velero → repository not initialized*; often the kopia repo or the Longhorn scheduling ceiling. |
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
