# Node Bootstrap — building or rebuilding a k3s server node

**Created:** 2026-07-03 (when the etcd-s3 config made the list of node-local
state long enough to deserve its own document).

Everything Kubernetes-side lives in Git and Flux rebuilds it. **This document
is the other half**: the node-local state that exists only on the Ubuntu VMs
themselves and must be re-applied by hand when a node is rebuilt from scratch.
Skipping any item here produces a node that *looks* healthy and then fails in
a documented, already-debugged way — each section links the incident.

The three servers (all control-plane + etcd):

| Node | IP | Notes |
|---|---|---|
| k3s-server-01 | 10.10.13.11 | Ubuntu 26.04.1 LTS, kernel 7.0.0-38 (upgraded from EOL 25.10 on 2026-09-23; package updates 2026-10-23) |
| k3s-server-02 | 10.10.13.12 | Ubuntu 26.04.1 LTS, kernel 7.0.0-38 (package updates 2026-10-23) |
| k3s-server-03 | 10.10.13.13 | Ubuntu 26.04.1 LTS, kernel 7.0.0-38 (package updates 2026-10-23) |

Upgrades: [maintenance.md → Ubuntu upgrades](maintenance.md#ubuntu-upgrades--one-node-at-a-time).

All run on Proxmox with `virtio_net` NICs and a second disk (`sdb`, 250 GiB)
for Longhorn at `/var/lib/longhorn`.

---

## 1. OS prerequisites

```bash
# Longhorn needs the iSCSI initiator
sudo apt-get install -y open-iscsi
sudo systemctl enable --now iscsid

# Longhorn disk: sdb formatted and mounted at /var/lib/longhorn (see
# maintenance.md "Growing the Longhorn disk" for the layout used).
```

## 2. multipathd blacklist — REQUIRED before Longhorn workloads land

Ubuntu's `multipathd` claims any SCSI device by default, including the
`/dev/sd*` devices Longhorn exposes over iSCSI. Without the blacklist, CSI
mounts fail with `already mounted or mount point busy` (2026-05-18 incident:
659+ pod restarts, 2 days of grafana-cnpg downtime).

`/etc/multipath.conf`:
```
blacklist {
    devnode "^sd[a-z]+"
}
```
```bash
sudo systemctl reload multipathd || sudo multipath -r
```

## 3. k3s install / rejoin

k3s v1.35.x, three-server embedded-etcd topology. New cluster: install the
first server with `--cluster-init`; join the others with `--server
https://10.10.13.11:6443` and the cluster token. **Write the config file
(section 4) BEFORE installing/starting k3s** so the node comes up with the
right flags from the first boot.

Replacing a dead node in an existing cluster: remove the old etcd member
first (`kubectl delete node <name>`, then check
`kubectl -n kube-system get etcdsnapshotfiles` / k3s docs on member removal),
then join the rebuilt node with the same name and IP.

## 4. `/etc/rancher/k3s/config.yaml` — the canonical content

Root-only (`chmod 600` — it contains the B2 key). Assembled from three
changes over time; a rebuilt node needs all of them:

```yaml
# --- TLS SANs (pre-existing) ---
tls-san:
  - 10.10.13.11        # this node's IP + any names used to reach the API

# NOTE: graceful node shutdown is deliberately NOT here as kubelet-arg.
# The k3s.service command line already passes --kubelet-arg, and a key set on
# the command line replaces the config.yaml value, so it was silently dropped
# on all three nodes. It lives in a kubelet drop-in instead (section 4a).

# --- etcd snapshots -> B2 (P1.2, 2026-07-03) ---
# Key is scoped to the k3s-safeqbit-etcd bucket ONLY (blast-radius
# separation from the Velero credentials). Key lives here and in
# Vaultwarden — never in Git.
etcd-s3: true
etcd-s3-endpoint: s3.us-east-005.backblazeb2.com
etcd-s3-region: us-east-005
etcd-s3-bucket: k3s-safeqbit-etcd
etcd-s3-folder: snapshots
etcd-s3-access-key: <keyID from Vaultwarden>
etcd-s3-secret-key: <applicationKey from Vaultwarden>
etcd-snapshot-retention: 8
etcd-snapshot-schedule-cron: "0 0 * * 0"   # weekly, Sunday 00:00 UTC
```

**Not everything is in this file.** On the existing nodes, `--server`,
`--node-ip`, `--advertise-address`, most `--tls-san`s, the node labels,
`--disable traefik/servicelb/local-storage` and the `--kubelet-arg`
`system-reserved`/`kube-reserved` pair come from the `k3s.service` command
line. Anything set there wins over this file. To see what k3s actually
started with:
`kubectl get node $NODE -o jsonpath='{.metadata.annotations.k3s\.io/node-args}'`.

Apply config changes with `sudo systemctl restart k3s` — **one node at a
time**, waiting for `kubectl get nodes` Ready + `kubectl get --raw /readyz`
between nodes (etcd quorum: never restart two servers at once).

## 4a. Graceful node shutdown — two drop-ins

Both files are required; either one alone does nothing useful. The why, and
the apply/verify runbook, are in
[maintenance.md → Graceful Node Shutdown](maintenance.md#graceful-node-shutdown-kubelet).

`/var/lib/rancher/k3s/agent/etc/kubelet.conf.d/50-graceful-shutdown.conf`
(k3s-supported kubelet drop-in; kubelet reads it at k3s start):
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
shutdownGracePeriod: 120s
shutdownGracePeriodCriticalPods: 30s
```

`/etc/systemd/logind.conf.d/zz-k3s-graceful-shutdown.conf`:
```ini
[Login]
InhibitDelayMaxSec=130
```
The `zz-` prefix matters. Ubuntu 26.04 ships
`/usr/lib/systemd/logind.conf.d/unattended-upgrades-logind-maxdelay.conf`
(`InhibitDelayMaxSec=30`), and the last drop-in by filename wins. `10-…` and
`99-…` both sort before it and are silently overridden. Without the cap
raised, logind cuts kubelet's 120s shutdown off after 30s.

## 5. flannel checksum-offload fix — REQUIRED on Proxmox virtio_net

After reboot, `flannel.1` can come up with `tx-checksum-ip-generic: on`,
which corrupts VXLAN-carried TCP/UDP. ICMP works (deceptively), everything
else across nodes times out (2026-05-27 incident). Install the persistent
unit:

`/etc/systemd/system/flannel-fix-offload.service` — full unit content in
[maintenance.md → Flannel VXLAN Checksum Offload](maintenance.md#flannel-vxlan-checksum-offload-post-reboot).

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now flannel-fix-offload.service
```

## 6. What you deliberately do NOT need to do

- **CoreDNS replicas** — do NOT edit
  `/var/lib/rancher/k3s/server/manifests/coredns.yaml` (the old approach;
  k3s reverts it). The Flux-managed cluster-proportional-autoscaler
  (`configs/coredns-autoscaler.yaml`) enforces 3 replicas continuously.
- **Anything application-level** — Flux reconciles the entire platform and
  all apps from this repo once the node joins.

## 7. Post-bootstrap verification

```bash
kubectl get nodes                                   # Ready
kubectl get --raw /readyz                           # ok
kubectl -n kube-system get deploy coredns           # 3/3 within ~1 min
systemctl status flannel-fix-offload.service        # active (exited)
systemctl show k3s -p Wants --value | grep -o flannel-fix-offload.service  # wired to k3s
ethtool -k flannel.1 | grep tx-checksum-ip-generic  # off
grep -c blacklist /etc/multipath.conf               # >= 1
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager InhibitDelayMaxUSec   # t 130000000
kubectl get --raw "/api/v1/nodes/$NODE/proxy/configz" | grep -o '"shutdownGracePeriod[^,]*'
                                                    # 2m0s and 30s
sudo k3s etcd-snapshot save                         # then:
kubectl get etcdsnapshotfiles | grep s3://          # upload visible
```

---

## Full cluster rebuild (all nodes lost)

Two paths, depending on what survived:

**Path A — etcd snapshot exists (B2 `k3s-safeqbit-etcd`): restore state.**
1. Bootstrap ONE node through sections 1–5 (don't join others yet).
2. Restore cluster state:
   ```bash
   k3s server --cluster-reset \
     --cluster-reset-restore-path=<snapshot-name> \
     --etcd-s3 --etcd-s3-endpoint=s3.us-east-005.backblazeb2.com \
     --etcd-s3-region=us-east-005 --etcd-s3-bucket=k3s-safeqbit-etcd \
     --etcd-s3-folder=snapshots \
     --etcd-s3-access-key=<keyID> --etcd-s3-secret-key=<appKey>
   ```
   This brings back **everything** — all objects, PVC bindings, and the
   sealed-secrets private key. Flux resumes on its own.
3. Bootstrap and join the other two nodes.
4. Longhorn volume *data* is gone with the disks — restore per-app from
   Velero/B2 ([backup-strategy.md → Restoring](backup-strategy.md#restoring));
   NFS-backed data reconnects per the NFS restore procedure there.

**Path B — no etcd snapshot: rebuild from Git.**
1. Bootstrap all three nodes (sections 1–5).
2. Restore the sealed-secrets master key from the offline backup
   (Vaultwarden) **before** apps reconcile, or every SealedSecret fails.
3. `flux bootstrap github` against this repo.
4. Velero restores volume data per namespace; NFS data per the
   backup-strategy NFS procedure; CNPG databases from Velero-restored
   snapshots or CNPG backups.

Path A is strictly better when available — it is the reason the etcd-s3
layer exists.
