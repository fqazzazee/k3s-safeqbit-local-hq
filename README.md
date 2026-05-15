# safeqbit-local-hq — Kubernetes Cluster

![K3s](https://img.shields.io/badge/K3s-FFC61C?style=for-the-badge&logo=k3s&logoColor=black)
![Flux](https://img.shields.io/badge/Flux-5468FF?style=for-the-badge&logo=flux&logoColor=white)
![Longhorn](https://img.shields.io/badge/Longhorn-5F224E?style=for-the-badge&logo=longhorn&logoColor=white)
![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Let's Encrypt](https://img.shields.io/badge/Let's%20Encrypt-003A70?style=for-the-badge&logo=letsencrypt&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![CloudNativePG](https://img.shields.io/badge/CloudNativePG-336791?style=for-the-badge&logo=postgresql&logoColor=white)
![Vaultwarden](https://img.shields.io/badge/Vaultwarden-175DDC?style=for-the-badge&logo=vaultwarden&logoColor=white)
![Authentik](https://img.shields.io/badge/Authentik-FD4B2D?style=for-the-badge&logo=authentik&logoColor=white)
![Immich](https://img.shields.io/badge/Immich-4250AF?style=for-the-badge&logo=immich&logoColor=white)
![PhotoPrism](https://img.shields.io/badge/PhotoPrism-7B4FFF?style=for-the-badge&logo=photoprism&logoColor=white)
![NetBox](https://img.shields.io/badge/NetBox-006F94?style=for-the-badge&logo=netbox&logoColor=white)
![Uptime Kuma](https://img.shields.io/badge/Uptime%20Kuma-5CDD8B?style=for-the-badge&logo=uptimekuma&logoColor=black)
![AFFiNE](https://img.shields.io/badge/AFFiNE-1E1E1E?style=for-the-badge&logo=affine&logoColor=white)
![Passzilla](https://img.shields.io/badge/Passzilla-3D6EB4?style=for-the-badge&logoColor=white)

A production-shaped homelab Kubernetes cluster running on three Proxmox-hosted nodes,
managed entirely through GitOps. Built and operated by Fadi Qazzazee.

---

## Overview

A 3-node K3s cluster with embedded etcd, deployed across three physical Proxmox hosts
for high availability. All nodes serve as both control plane and workers. The cluster
hosts a range of self-hosted services behind a full platform layer: Layer 2 load
balancing, automatic TLS termination, distributed block storage, NFS-backed storage,
encrypted secrets management, continuous monitoring, and full GitOps-driven
reconciliation via Flux.

The design goal is a cluster that rebuilds completely from three inputs: a fresh K3s
install, a Git repository, and a single encryption key secured stored in my password manager. No manual imperative steps,
no undocumented state.

---

## Infrastructure

| Item | Value |
|---|---|
| Cluster name | safeqbit-local-hq |
| Distribution | K3s, embedded etcd |
| LAN domain | local.safeqbit.com |
| Cluster domain | cluster.local |
| Cluster VLAN | 10.10.13.0/24 |
| API endpoint | k3s.local.safeqbit.com (round-robin, all 3 nodes) |

---

## Nodes

All three nodes run Proxmox VE and host K3s server VMs. Every node participates in
etcd quorum and runs workloads — there are no dedicated worker-only nodes.

| Node | Host | Hardware | CPU | RAM | Network |
|---|---|---|---|---|---|
| k3s-server-01 | Meatball | Minisforum BD790i X3D Custom Build Proxmox Node | 32 vCPU | 96 GB DDR5 | Dual 10 Gb Fiber |
| k3s-server-02 | Dumpling | AOOSTAR R7 Mini PC Proxmox Node | 16 vCPU | 64 GB DDR4 | Dual 2.5 GbE |
| k3s-server-03 | Omen | HP Omen 15T w/ RTX2070 Notebook Repurposed as a Proxmox Node + vPBS | 12 vCPU | 32 GB DDR4 | 1 GbE |

### Node labels

| Node | tier | zone |
|---|---|---|
| k3s-server-01 | primary | meatball |
| k3s-server-02 | secondary | dumpling |
| k3s-server-03 | tertiary | Omen |

Workloads carry a soft scheduling preference toward `tier=primary`. The scheduler
lands work on Meatball by default and falls back to the other nodes freely — no hard
restrictions.

---

## Storage

The cluster runs a two-tier storage architecture. Longhorn provides distributed block
storage replicated across cluster nodes for performance-sensitive and HA workloads.
TrueNAS provides NFS-backed storage for workloads that require shared or bulk access.

### Longhorn - primary distributed block storage

Longhorn is the default StorageClass for the cluster. It provisions replicated block
volumes directly on node-local disks, with data distributed across nodes for resilience.
Volume snapshots and incremental backups are managed by Longhorn's built-in backup engine.

- Volumes are replicated across multiple nodes in-cluster
- Snapshot and backup support via the Longhorn UI and CSI interface
- Integrated with Velero via the CSI snapshot API for coordinated cluster-level backups

### TrueNAS - secondary NFS storage

TrueNAS provides NFS-backed storage for workloads with shared access requirements or
where bulk capacity is preferred over distributed block performance.

| Property | Value |
|---|---|
| Primary NAS |
| Pool | 2TB NVME |
| Dataset | nvme2tb/k8s |
| NFS export | /mnt/nvme2tb/k8s |
| Network authorization | Cluster VLAN only restricted by TrueNAS ACL and Firewall Policy|
| StorageClass | nfs-truenas |

Dynamic provisioning is handled by `nfs-subdir-external-provisioner`. PVC creation
automatically provisions a dedicated subdirectory on TrueNAS — no manual NFS export
management per workload.

### TrueNAS - bulk media storage

A second NFS provisioner serves the bulk media library on a separate TrueNAS dataset
backed by a spinning-disk array. Immich and Photoprism access their existing media
directories via static PVs pointing directly to pre-existing paths on this share —
no data migration required. New workloads that need bulk media capacity use dynamic
provisioning via the `nfs-bulk-media` StorageClass (RWX, Retain policy).

### TrueNAS replication target

A secondary TrueNAS server with 4 TB of capacity serves as the replication target for
the primary NAS. Replication runs every 30 minutes via SSH, providing a continuously
updated offsite-ready copy of all NAS datasets.

| Property | Value |
|---|---|
| Replication target | Secondary TrueNAS server |
| Capacity | 4 TB NVME |
| Replication interval | Every 30 minutes |
| Method | SSH + NETCAT snapshot replication |
| Failover | Manual (documented runbook) |

---

## Backups

Cluster workload backups are handled by **Velero**, with two complementary targets.

### Velero - cluster backup and restore

Velero performs scheduled backups of Kubernetes resource state and persistent volume
data. It integrates with Longhorn via the CSI snapshot API to capture application-consistent
volume snapshots before shipping data offsite.

| Property | Value |
|---|---|
| Tool | Velero |
| Volume snapshot driver | CSI (Longhorn VolumeSnapshotClass) |
| Offsite backup target | Backblaze B2 |
| Backup scope | Namespaced resources + PVC data |

Backups are stored in a dedicated Backblaze B2 bucket, providing durable offsite retention
independent of the local infrastructure. Credentials for B2 access are managed as
SealedSecrets in Git.

### Backup summary

| Layer | What it protects | Where it goes |
|---|---|---|
| Longhorn snapshots | In-cluster volume point-in-time recovery | Local (node disks) |
| Velero + B2 | Full workload state and volume data | Backblaze B2 (offsite) |
| TrueNAS replication | NFS dataset contents | Secondary TrueNAS (LAN) |

---

## Platform components

### MetalLB - Layer 2 load balancing

Assigns real LAN IPs to `LoadBalancer`-type Services without a cloud provider.
Operates in Layer 2 mode (ARP-based, no BGP).

| Pool | Range | Assignment |
|---|---|---|
| reserved-pool | 10.10.13.50-59 | Manual via annotation |
| auto-pool | 10.10.13.60-100 | Automatic (default) |

### ingress-nginx - HTTP/HTTPS ingress

Deployed as a DaemonSet across all nodes with `externalTrafficPolicy: Local`, preserving
real client IPs through the load balancer. A single VIP at 10.10.13.50 serves all
cluster ingress. Individual services are routed by hostname via `Ingress` objects.

### cert-manager - automated TLS

Manages the full lifecycle of TLS certificates using Let's Encrypt. All challenges
use the DNS-01 method via the Cloudflare API, which allows internal hostnames to
obtain valid public certificates without requiring inbound HTTP access. Two issuers
are maintained: staging (for testing) and production.

### Sealed Secrets - encrypted secret management

Enables secrets to be committed to Git safely. Secrets are encrypted on the workstation
using `kubeseal` and stored in the repository as `SealedSecret` objects. Decryption
happens exclusively in-cluster via the sealed-secrets controller. The master encryption
key is stored offline in an encrypted password manager.

### CloudNativePG - PostgreSQL operator

Manages all PostgreSQL databases in the cluster. Deployed in `cnpg-system` with 2 replicas
(pod anti-affinity across nodes) and watches all namespaces. Each workload that requires
Postgres declares its own `Cluster` CR in its own namespace — CNPG reconciles them all.

CNPG automatically provisions three Services per cluster (`-rw`, `-ro`, `-r`) for write,
read-only, and round-robin-read access. Credentials are generated per cluster into a
sealed secret. Backups use VolumeSnapshot via the Longhorn CSI interface, issuing a
`CHECKPOINT` before snapshotting to guarantee a clean, application-consistent state.

| Workload | Namespace | Instances | Notes |
|---|---|---|---|
| Authentik | `authentik` | 2 | SSO backend — hot standby for fast failover |
| Grafana | `monitoring` | 1 | Migrated from SQLite/NFS (deadlock issue) |
| NetBox | `netbox` | 1 | — |
| AFFiNE | `affine` | 1 | — |

### kube-prometheus-stack - observability

Full-stack cluster observability: Prometheus for metrics collection and Grafana for
visualization. Scrapes platform components (ingress-nginx, cert-manager, sealed-secrets,
node-exporter, kube-state-metrics) via ServiceMonitors. Preloaded dashboards cover
cluster-wide resource usage, per-namespace workloads, and node-level OS metrics.

---

## GitOps - Flux v2

The cluster is fully GitOps-managed via **Flux v2**, bootstrapped against a private
GitHub repository. All platform changes and workload deployments are made through Git.
Direct imperative operations (`helm install`, `kubectl apply`) are not used for
ongoing management.

### Repository structure

```
k3s-safeqbit-local-hq/
├── clusters/
│   └── safeqbit-local-hq/
│       ├── flux-system/                  # Flux bootstrap manifests
│       ├── apps.yaml                     # Kustomization: workloads
│       ├── infrastructure-controllers.yaml
│       └── infrastructure-configs.yaml
├── apps/
│   └── safeqbit-local-hq/               # Per-application manifests
└── infrastructure/
    └── safeqbit-local-hq/
        ├── controllers/                  # HelmReleases for all platform charts
        └── configs/                      # CRs that depend on those charts
                                          # (MetalLB pools, ClusterIssuers,
                                          #  SealedSecrets, VolumeSnapshotClass)
```

The `controllers/configs` split enforces dependency ordering. Flux `dependsOn`
ensures that operator CRDs exist before dependent custom resources are applied.
This matters most during a cluster rebuild, where everything comes up in a single
reconciliation pass.

### Disaster recovery

1. Fresh K3s install across the three nodes (same IPs, same node labels)
2. Restore the sealed-secrets master key into `kube-system` from offline backup
3. Run `flux bootstrap github` pointing at the repository
4. Flux reconciles in order: controllers, configs, workloads
5. Velero restores persistent volume data from Backblaze B2 for any workload that requires it
6. Cluster returns to the state committed in Git

The only state outside Git is the sealed-secrets master key (offline backup),
Longhorn volume data (backed up to B2 via Velero), and TrueNAS NFS data
(replicated to secondary).

---

## Workloads

### Self-hosted services

| Workload | Category |
|---|---|
| Vaultwarden | Password management |
| Authentik | Identity provider / SSO |
| Immich | Photo and video backup |
| PhotoPrism | Photo management and AI tagging |
| NetBox | Network documentation and IPAM |
| Uptime Kuma | Uptime and availability monitoring |
| Grafana | Metrics dashboards and visualization |
| Prometheus | Metrics collection and alerting |
| AFFiNE | Collaborative knowledge base |
| Passzilla | One-time secret and password sharing |

### Custom and development workloads

| Workload | Description |
|---|---|
| [NetScan](https://github.com/fqazzazee/NetScan-WoL) | Network scanning and Wake-on-LAN management |

Custom workloads are developed and iterated on-cluster using the same GitOps pipeline
as production services, providing a realistic deployment environment for internal tools.

---

## Security posture

- **Secrets** are never stored in plaintext in Git. All sensitive values are sealed
  with `kubeseal` and committed as `SealedSecret` objects.
- **TLS** is enforced on all exposed services via Let's Encrypt certificates managed
  by cert-manager. No self-signed certificates in any production path.
- **Master key** for secret decryption is stored offline in an encrypted password
  manager and never committed to the repository.
- **Network isolation** restricts NFS access to the cluster VLAN only. The cluster
  VLAN is segmented from other infrastructure segments.
- **Image provenance** follows standard OCI conventions. Container images are pulled
  from public registries and pinned to explicit tags or digests per workload.

---

## Technology summary

| Layer | Technology |
|---|---|
| Hypervisor | Proxmox VE |
| Kubernetes distribution | K3s |
| CNI | Flannel (K3s default) |
| Container runtime | containerd |
| Load balancing | MetalLB (Layer 2) |
| Ingress | ingress-nginx |
| Certificate management | cert-manager + Let's Encrypt (DNS-01 / Cloudflare) |
| Secret management | Sealed Secrets (Bitnami) |
| Primary storage | Longhorn (distributed block, CSI) |
| Secondary storage | nfs-subdir-external-provisioner + TrueNAS NFS (k8s workloads) |
| Bulk media storage | nfs-subdir-external-provisioner + TrueNAS NFS (media library) |
| Database operator | CloudNativePG (CNPG) |
| NAS replication | TrueNAS to TrueNAS (SSH, 30-minute interval) |
| Backup | Velero + Backblaze B2 |
| Observability | kube-prometheus-stack (Prometheus + Grafana) |
| GitOps | Flux v2 |
| Package management | Helm 3 |
| DNS | Cloudflare (external), local.safeqbit.com (internal) |
