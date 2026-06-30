# safeqbit-local-hq - my self-hosted Kubernetes home cluster

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
![Pulse](https://img.shields.io/badge/Pulse-00B4D8?style=for-the-badge&logoColor=white)

> A small private cloud running in my home - three computers working together to host
> my passwords, photos, notes, and a handful of other apps I'd rather not rent from a
> big tech company. Everything here is defined in this repository, so the whole thing
> can rebuild itself from scratch. Built and run by **Fadi Qazzazee**.

---

## 👋 What is this, in plain English?

I self-host. Instead of paying monthly for a password manager, a photo backup, a notes
app, and so on, I run my own copies at home and keep control of my data.

The challenge: if you run all of that on **one** computer and that computer dies - a
failed disk, a bad reboot, a power blip - *everything* goes down at once, and you might
lose data you can't get back.

So instead of one computer, I run **three**, wired together so they behave like a single
system. That system is called a **cluster**. If one machine fails, the other two keep
your apps running and your data safe. The whole setup is described in plain text files
in this repo, and a tool called **Flux** continuously makes the cluster match what's
written here. Change a file, commit it, and the cluster updates itself. Lose a machine,
fix it, and it rejoins automatically.

And honestly, that's only half the reason this exists. The other half is **learning**.
I could run all of these apps the easy way - one server, a handful of containers, done.
Instead I deliberately built it the hard way: real Kubernetes, real GitOps, real
high-availability storage and databases, the same tools and patterns used in production
environments at work. This repo is my hands-on classroom for Kubernetes, Flux, storage,
networking, and operations - and keeping it documented well enough for a stranger to
follow is part of that exercise. If you're learning this stuff too, I hope it's useful.

If some of the words below are new to you, here's the quick version:

| Term | What it actually means |
|---|---|
| **Cluster** | A group of computers that pool their resources and act as one. |
| **Node** | A single computer in the cluster. I have three. |
| **Kubernetes / K3s** | The "operating system" for the cluster - it decides which app runs on which node, restarts crashed apps, and moves work off a node that dies. K3s is a lightweight version of Kubernetes. |
| **Container / Pod** | An app packaged up so it runs the same anywhere. A pod is one or more containers running together. |
| **GitOps / Flux** | "Git is the single source of truth." Everything the cluster should run lives in this Git repo; Flux watches the repo and applies it. No clicking around, no undocumented changes. |

---

## 🧱 Why three machines instead of one?

This is the whole point of the project, so it's worth spelling out. Here's the same set
of apps run two ways:

| | One server (the usual way) | This cluster (three nodes) |
|---|---|---|
| A machine reboots or crashes | **Everything is offline** until it's back | Apps **move to the other two nodes** and keep running |
| A disk fails | Possible **permanent data loss** | Data is **replicated to other nodes**, plus off-site backups |
| You want to do maintenance | You schedule downtime | Drain one node, work on it, others carry the load - **no downtime** |
| Need more capacity | Buy a bigger box, migrate everything | **Add another node** and the cluster spreads work onto it |
| "Where is everything configured?" | In your head, and scattered config files | **In this Git repo**, all of it |

The cluster gets its resilience from a few layers stacked together:

- **Three nodes** Each Node helps run apps *and*
  helps make decisions. They use a voting system (called *etcd quorum*) so that as long
  as two of the three are alive, the cluster keeps making decisions and stays online.
- **Apps reschedule themselves.** If a node goes away, Kubernetes notices and restarts
  that node's apps on the survivors, automatically.
- **Data is replicated.** Storage (Longhorn) keeps copies of each app's data on more
  than one node, so losing a node doesn't lose the data.
- **The important databases run as a pair.** A primary and a standby on *different*
  nodes, so a single machine failure never takes both copies of a database.
- **Backups on top of all that.** Even if the whole site burned down, the data is also
  copied off-site (more on that below).

No setup is bulletproof, but the goal is simple: **no single thing should be able to
take everything down or destroy my data.**

---

## ⚙️ How it works (the 60-second tour)

```
        I edit a file              Flux notices            Kubernetes makes
        and push to Git    ─────►  the change in    ─────► it real on the
                                   this repo               three nodes
```

Everything the cluster runs - every app, every setting, every database - is written down
in this repository as plain text. **Flux** runs inside the cluster, watches this repo,
and continuously reconciles reality against what's committed. I almost never run commands
directly against the cluster; I change a file, and the cluster catches up.

That design buys two things that matter:

1. **The repo *is* the documentation.** Nothing important lives only in my head or in a
   one-off command I ran six months ago.
2. **The cluster can rebuild itself.** From three blank machines, it comes back from just
   three inputs: a fresh K3s install, this Git repo, and one encryption key I keep in my
   password manager. No manual clicking, no undocumented state.

When someone visits one of my apps, the request flows like this:

```
Visitor ─► Cloudflare / my LAN ─► ingress (front door) ─► the app's pod ─► its database
```

---

## 🖥️ The hardware

Three physical machines, each running [Proxmox](https://www.proxmox.com/) (a hypervisor)
which in turn runs a K3s virtual machine. All three are full members - there are no
"lesser" worker-only nodes; every node both runs apps and takes part in the cluster's
decision-making.

| Node | Nickname | Machine | CPU | RAM | Network |
|---|---|---|---|---|---|
| k3s-server-01 | **Meatball** | Minisforum BD790i X3D custom build | 32 vCPU | 96 GB DDR5 | Dual 10 Gb fiber |
| k3s-server-02 | **Dumpling** | AOOSTAR R7 mini PC | 16 vCPU | 64 GB DDR4 | Dual 2.5 GbE |
| k3s-server-03 | **Omen** | HP Omen 15T laptop (RTX 2070) repurposed as a node | 12 vCPU | 32 GB DDR4 | 1 GbE |

The nodes carry descriptive labels (`tier` and `zone`), but I don't pin apps to specific
machines. The only scheduling rule in play is *spread the copies out*: the high-availability
platform pieces and the paired databases use **soft anti-affinity** so their two replicas
land on different nodes. That way one machine failing never takes out both halves of a pair.
Everything else, Kubernetes places wherever it fits.

### Networking

The three nodes don't just plug into the home router - they sit on a purpose-built
network designed the same way you'd lay one out in a small datacenter.

| Gear | Role |
|---|---|
| **MikroTik CRS310-8G+2S+IN** | Homelab/Core L3 switch between the three nodes (8× 2.5 GbE + 2× 10G SFP+). The 10G fiber and 2.5 GbE node links all terminate here. |
| **UniFi Cloud Gateway Max (UCG-MAX)** | Router, firewall, and inter-VLAN gateway for the whole network. |

A few deliberate choices, and *why*:

- **VLAN segmentation.** The cluster lives on its own VLAN (`10.10.13.0/24`), separated
  from trusted client devices, IoT, and management. Each segment is a separate broadcast
  domain with its own firewall policy, so a compromised device in one zone can't freely
  reach another. This is classic network micro-segmentation / least-privilege.
- **Router-on-a-stick.** VLANs are trunked from the switch up to the UCG-MAX, which does
  the routing *between* them. All inter-VLAN traffic passes through one place where
  firewall rules are enforced - east-west traffic isn't implicitly trusted.
- **LACP link aggregation.** The two nodes with dual NICs - Meatball (dual 10G fiber) and
  Dumpling (dual 2.5 GbE) - are bonded to the switch with LACP (IEEE 802.3ad). That gives
  both more throughput *and* link-level redundancy: a cable or port can fail and the node
  stays connected. Omen runs a single 1 GbE link. The 10G fiber pair lands on the switch's
  SFP+ ports; the 2.5 GbE pair on the copper ports.

The payoff: storage replication and database traffic between nodes ride 10G/2.5G bonded
links instead of fighting for the home network, and the cluster is firewalled off from
everything else by default.

### Cluster facts

| Item | Value |
|---|---|
| Cluster name | safeqbit-local-hq |
| Distribution | K3s (lightweight Kubernetes), embedded etcd |
| Nodes | 3 (all control-plane + worker) |
| LAN domain | local.safeqbit.com |
| Cluster VLAN | 10.10.13.0/24 (segmented from the rest of the network) |
| API endpoint | k3s.local.safeqbit.com (round-robin across all 3 nodes) |

---

## 💾 Storage - where the data lives

Different apps want different things from storage, so there are two tiers.

### Longhorn: fast, replicated storage for apps

[Longhorn](https://longhorn.io/) is the default storage for the cluster. When an app
needs a disk, Longhorn carves one out of the nodes' local drives and **keeps replicated
copies on more than one node**. If a node dies, the data is still there on another. This
is where databases and performance-sensitive apps live.

- Replicated across multiple nodes inside the cluster
- Point-in-time snapshots via the Longhorn UI or the CSI interface
- Wired into Velero (the backup tool) for coordinated, cluster-wide backups

### TrueNAS: shared and bulk storage over NFS

For things that need a lot of space or shared access (like a media library), the cluster
mounts storage from a [TrueNAS](https://www.truenas.com/) box over the network (NFS).

| Property | Value |
|---|---|
| Primary NAS | TrueNAS |
| Pool | 2 TB NVMe |
| Dataset | nvme2tb/k8s |
| NFS export | /mnt/nvme2tb/k8s |
| Access | Cluster VLAN only, enforced by TrueNAS ACLs + firewall policy |
| StorageClass | `nfs-truenas` |

A provisioner (`nfs-subdir-external-provisioner`) hands out a dedicated folder on the NAS
automatically whenever an app asks for one - no manual NFS export juggling.

**Bulk media:** a second NFS tier serves a large spinning-disk array. Immich and
PhotoPrism point straight at my existing photo/video folders there, so there was no data
migration when I moved them onto the cluster. New apps that need bulk space use the
`nfs-bulk-media` storage class.

**Replication:** a second TrueNAS (4 TB NVMe) pulls a copy of the primary every 30 minutes
over SSH, giving me a continuously-updated, ready-to-go second copy of all NAS data.

---

## 🛟 Backups - three copies, three failure modes

A backup is only useful for the disaster it was designed for, so I run three layers, each
covering a different "what if."

| Layer | Tool | Protects against | Where it goes |
|---|---|---|---|
| 1 | **Longhorn snapshots** | "Oops, I deleted that an hour ago" | On the volume itself (in-cluster) |
| 2 | **CNPG database snapshots** | A database corrupting or a disk being lost on a node | Longhorn (in-cluster, app-consistent) |
| 3 | **Velero → Backblaze B2** | The whole cluster or site being lost | Off-site object storage |
| + | **TrueNAS → TrueNAS** | The primary NAS dying | A second NAS on the LAN |

A few details for the curious:

- **Layer 2** takes a clean, application-consistent snapshot of each PostgreSQL database
  daily/weekly (it tells the database to flush first). A daily cleanup job prunes old ones.
- **Layer 3** (Velero) ships both the Kubernetes configuration *and* the volume data up to
  a [Backblaze B2](https://www.backblaze.com/cloud-storage) bucket off-site. Schedules are
  staggered so only one app uploads per day, keeping costs predictable. Credentials for B2
  are stored encrypted in this repo (see *Security* below).

---

## 🧩 The platform layer - the plumbing that makes it a "cloud"

These are the behind-the-scenes components. None of them are apps I use directly; they're
the services that turn three computers into something that behaves like a cloud provider.

### MetalLB: gives apps real network addresses

In a real cloud, asking for a load balancer gets you a public IP. At home there's no cloud
to ask, so [MetalLB](https://metallb.io/) hands out IPs from my LAN instead.

| Pool | Range | Assignment |
|---|---|---|
| reserved-pool | 10.10.13.50-59 | Manual (by annotation) |
| auto-pool | 10.10.13.60-100 | Automatic (default) |

### ingress-nginx: the front door

All web traffic enters through a single address (`10.10.13.50`) and
[ingress-nginx](https://kubernetes.github.io/ingress-nginx/) routes each request to the
right app based on its hostname. It runs on every node and preserves the real visitor IP.

### cert-manager: automatic HTTPS

[cert-manager](https://cert-manager.io/) gets and renews real
[Let's Encrypt](https://letsencrypt.org/) TLS certificates automatically, even for
internal-only hostnames, by proving domain ownership through Cloudflare's DNS API. No
expired-certificate warnings, no manual renewals, no self-signed certs.

### Sealed Secrets: how passwords live safely in a public repo

Secrets (database passwords, API tokens) obviously can't sit in plain text in a public
Git repo. [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) solves this: I
encrypt each secret on my laptop with `kubeseal`, and only the controller *inside my
cluster* can decrypt it. The encrypted blobs are safe to commit and useless to anyone
else. The master key that could decrypt them never touches the repo - it lives offline in
my password manager.

### CloudNativePG: managed databases

[CloudNativePG](https://cloudnative-pg.io/) (CNPG) runs the PostgreSQL databases. Each app
that needs one gets its own database cluster, and the important ones run as a
**primary + standby pair on different nodes** so a single machine failure doesn't take the
database offline.

| Database | App | Instances | Notes |
|---|---|---|---|
| authentik | Authentik | 2 | Hot standby for fast failover |
| grafana-cnpg | Grafana | 2 | Moved off SQLite (which deadlocked); primary + standby |
| netbox | NetBox | 2 | Primary + standby |
| affine | AFFiNE | 2 | Primary + standby |

All four pairs use anti-affinity so the primary and standby never share a node. (Immich
and PhotoPrism run their own tuned PostgreSQL; Vaultwarden uses a simple SQLite file on
replicated storage.)

### cloudflared: secure access from outside the house

A few apps need to be reachable from the public internet. Rather than opening ports on my
router (and painting a target on my home network), I use
[Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/):
small connectors run *inside* the cluster and dial **outward** to Cloudflare. Traffic comes
back down that outbound connection. No open inbound ports, nothing exposed directly. Each
tunnel runs two copies for redundancy.

### kube-prometheus-stack: monitoring and alerts

[Prometheus](https://prometheus.io/) collects metrics from everything;
[Grafana](https://grafana.com/) draws the dashboards. If something breaks - a node, an app,
or a tunnel - Alertmanager pings me on Slack. This is how I find out about problems before
they become outages.

---

## 🔁 GitOps & disaster recovery

The cluster is managed entirely with [Flux v2](https://fluxcd.io/), bootstrapped against
this GitHub repo over SSH. Every change goes through Git; I don't run `helm install` or
`kubectl apply` by hand for ongoing work.

### Repository layout

```
k3s-safeqbit-local-hq/
├── clusters/safeqbit-local-hq/
│   ├── flux-system/                  # Flux's own bootstrap manifests
│   ├── apps.yaml                     # tells Flux where the apps are
│   ├── infrastructure-controllers.yaml
│   └── infrastructure-configs.yaml
├── apps/safeqbit-local-hq/           # one folder per app I run
└── infrastructure/safeqbit-local-hq/
    ├── controllers/                  # the platform pieces (Helm charts)
    └── configs/                      # settings that depend on those pieces
```

The controllers/configs split lets Flux bring things up **in the right order** during a
rebuild - operators first, then the things that depend on them - all in one pass.

### Rebuilding from nothing

If all three machines were wiped tomorrow, recovery is a short list:

1. Fresh K3s install on the three nodes (same IPs, same labels).
2. Restore the one sealed-secrets master key from my offline backup.
3. Run `flux bootstrap github` pointed at this repo.
4. Flux reconciles everything in order: platform → config → apps.
5. Velero restores volume data from Backblaze B2 for apps that need it.
6. The cluster is back to exactly what's committed here.

The only things that live *outside* Git are: the master key (offline), Longhorn volume data
(backed up to B2), and the NAS data (replicated to the second NAS).

---

## 🔐 Security posture

Each control below is a deliberate choice, not a default. I've noted *why* it's done that
way and the recognised standard or framework it maps to - partly for anyone reading, and
partly because reasoning from established guidance is itself one of the things I'm here to
learn.

- **No plaintext secrets, ever.** Every sensitive value is encrypted with `kubeseal` and
  committed as a `SealedSecret`; the only key that can decrypt them lives in-cluster.
  *Why:* secrets stored in a Git repo are secrets exposed forever in history - encrypting
  them at rest and separating the decryption key removes that exposure.
  *Maps to:* CIS Kubernetes Benchmark §5.4 (Secrets Management), NIST SP 800-57
  (key management), and the [Twelve-Factor App](https://12factor.net/config) config principle.
- **The decryption key stays offline.** The single master key that could unseal every
  secret never touches the repo - it lives in an encrypted password manager.
  *Why:* the security of everything sealed reduces to protecting one root of trust, so it's
  kept out-of-band and offline (defense in depth - no single online system can leak it).
  *Maps to:* NIST SP 800-57 key-custody guidance; the principle of protecting the root key
  separately from the data it protects.
- **Encryption in transit, everywhere.** Real Let's Encrypt certificates on every exposed
  service, renewed automatically; no self-signed certs in any live path.
  *Why:* unencrypted internal traffic can be intercepted or tampered with; valid certs also
  prevent users training themselves to click through warnings.
  *Maps to:* NIST SP 800-52 Rev. 2 (TLS guidelines) and the OWASP Transport Layer Security
  Cheat Sheet.
- **Network segmentation & least privilege.** The cluster sits on an isolated VLAN; storage
  (NFS) is reachable only from that VLAN, and inter-VLAN traffic is routed through a single
  firewalled gateway (the router-on-a-stick described above).
  *Why:* a flat network means one compromised device can reach everything. Segmenting
  contains blast radius and enforces least-privilege between zones.
  *Maps to:* CIS Controls v8 - Control 4 (Secure Configuration) and Control 12 (Network
  Infrastructure Management) - and NIST SP 800-207 (Zero Trust Architecture).
- **No open inbound ports - reduced attack surface.** Public access is via *outbound-only*
  Cloudflare Tunnels; nothing on the home network listens for unsolicited inbound traffic.
  *Why:* every open inbound port is a potential entry point. Outbound-only tunnels remove
  the exposed listening surface entirely.
  *Maps to:* the attack-surface-reduction principle in NIST SP 800-207 (Zero Trust) and
  OWASP's minimize-attack-surface guidance.
- **Pinned, provenanced images - supply-chain hygiene.** Containers come from public
  registries pinned to explicit tags or digests per workload.
  *Why:* pulling `:latest` means you can't reproduce what you ran, and a mutated upstream
  tag could swap code under you. Pinning makes deployments reproducible and auditable.
  *Maps to:* CIS Kubernetes Benchmark §5.5 (Image Provenance), NIST SP 800-190 (Application
  Container Security), and the [SLSA](https://slsa.dev/) supply-chain framework.

---

## 📦 Everything running on the cluster

### Apps I actually use

| App | What it's for |
|---|---|
| [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | Password manager (Bitwarden-compatible) |
| [Authentik](https://goauthentik.io/) | Single sign-on / identity provider |
| [Immich](https://immich.app/) | Self-hosted photo & video backup (Google Photos alternative) |
| [PhotoPrism](https://www.photoprism.app/) | Photo management with AI tagging |
| [NetBox](https://netbox.dev/) | Network documentation & IP address management |
| [AFFiNE](https://affine.pro/) | Notes & collaborative knowledge base |
| [Passzilla](https://github.com/pglombardo/PasswordPusher) | One-time secret / password sharing links |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Uptime & availability monitoring |
| [Apache Guacamole](https://guacamole.apache.org/) | Clientless HTML5 remote access (RDP/VNC/SSH) — SRA demo stack |
| [Pangolin](https://pangolin.net/) | Identity-aware remote access (HTTP + browser RDP/VNC/SSH) over WireGuard |
| [Grafana](https://grafana.com/) | Dashboards for everything the cluster reports |
| [Prometheus](https://prometheus.io/) | Metrics collection & alerting engine |
| [Pulse](https://github.com/rcourtman/Pulse) | Single-pane-of-glass monitoring for the Proxmox hosts, this k3s cluster, and the Docker host |

### Platform components (the plumbing)

| Component | Role |
|---|---|
| K3s + etcd | The cluster itself |
| Flux v2 | GitOps - keeps the cluster matching this repo |
| MetalLB | Hands out LAN IPs to services |
| ingress-nginx | Routes incoming web traffic |
| cert-manager | Automatic Let's Encrypt TLS |
| Sealed Secrets | Encrypted secrets safe to store in Git |
| CloudNativePG | Manages the PostgreSQL databases |
| Longhorn | Replicated block storage |
| NFS provisioners | Shared & bulk storage from TrueNAS |
| cloudflared | Outbound-only tunnels for public access |
| Velero | Off-site backups to Backblaze B2 |
| kube-prometheus-stack | Metrics, dashboards & alerts |

### Things I'm building

| Project | Description |
|---|---|
| [NetScan](https://github.com/fqazzazee/NetScan-WoL) | Network scanning & Wake-on-LAN management |

I develop and run my own tools on the same cluster and through the same GitOps pipeline as
everything else - a realistic place to test things in something close to production.

---

## Technology summary

<details>
<summary>Full stack, layer by layer (click to expand)</summary>

| Layer | Technology |
|---|---|
| Hypervisor | Proxmox VE |
| Kubernetes distribution | K3s |
| CNI | Flannel (K3s default) |
| Container runtime | containerd |
| Load balancing | MetalLB (Layer 2) |
| Ingress | ingress-nginx |
| Certificate management | cert-manager + Let's Encrypt (DNS-01 / Cloudflare) |
| External access | Cloudflare Tunnels (cloudflared, outbound-only) |
| Secret management | Sealed Secrets (Bitnami) |
| Primary storage | Longhorn (distributed block, CSI) |
| Secondary storage | nfs-subdir-external-provisioner + TrueNAS NFS (k8s workloads) |
| Bulk media storage | nfs-subdir-external-provisioner + TrueNAS NFS (media library) |
| Database operator | CloudNativePG (CNPG) |
| NAS replication | TrueNAS → TrueNAS (SSH, 30-minute interval) |
| Backup | Longhorn snapshots + CNPG VolumeSnapshot + Velero / Backblaze B2 |
| Observability | kube-prometheus-stack (Prometheus + Grafana + Alertmanager → Slack) |
| GitOps | Flux v2 |
| Package management | Helm 3 |
| DNS | Cloudflare (external), local.safeqbit.com (internal) |

</details>

---

## 🤖 A note on AI

Parts of this documentation were drafted and refined with the help of AI tools, and I use
AI as an assistant elsewhere in the project too. The architecture, decisions, hardware, and
running cluster are mine - AI helped me explain them more clearly and double-check the
details, not invent them. Anything stated here reflects the actual setup; if you spot
something that looks off, it's on me, so please open an issue.

---

## 🙏 Inspiration

A lot of the patterns here were shaped by [Techno Tim](https://technotim.com), whose
homelab and Kubernetes content got me started and kept me unstuck more than once. Highly
recommended if you're building something like this yourself.
