# Longhorn 1.11.3 → 1.12.1 — Held, and What It Needs

**Cluster:** safeqbit-local-hq
**Last reviewed:** 2026-09-22 (written after the upgrade was researched and deliberately held)

Longhorn stays on **1.11.3**. Everything about the upgrade was checked on
2026-09-22 and the only thing standing in the way is a decision, not a problem:
1.12 turns on internal NetworkPolicies that block Prometheus from scraping
Longhorn, and closing that gap means either adding one policy of our own or
declining upstream's new hardening. That call was deferred, so this doc holds
the research so none of it has to be redone.

> **One-way door.** Longhorn refuses to downgrade once 1.12.1 is applied. Take
> a Longhorn system backup before starting; there is no going back afterwards.

---

## 1. The blocker

1.12 ships six NetworkPolicies: `backing-image-data-source`,
`backing-image-manager`, `instance-manager`, `longhorn-manager`,
`longhorn-recovery-backend`, `longhorn-webhook`.

**They are not controlled by `networkPolicies.enabled`.** That value defaults to
`false` and is false for us, and reading it is how you talk yourself out of
looking further. The policies come from `networkPolicies.restrictInternalTraffic`,
which defaults to **`true`** and is an independent switch:

```yaml
networkPolicies:
  enabled: false                 # <- external access policies. off.
  restrictInternalTraffic: true  # <- THIS creates the six. on by default.
  type: "k3s"
```

`NetworkPolicy/longhorn-manager` sets `policyTypes: [Ingress]` on pods labelled
`app: longhorn-manager`, and every entry in its `from` list is a **podSelector
with no namespaceSelector** — so it admits same-namespace pods only
(longhorn-manager, longhorn-ui, longhorn-csi-plugin, recurring-job pods,
job-task pods, longhorn-driver-deployer).

Prometheus lives in `monitoring` and scrapes those pods directly on **:9500**:

```
ServiceMonitor monitoring/longhorn-manager
  namespaceSelector: longhorn-system
  selector:          app=longhorn-manager
  endpoint:          port "manager", /metrics, 30s
  → 3 targets, job "longhorn-backend", healthy on 1.11.3
```

It is not in the allow list. **The upgrade drops all Longhorn metrics silently** —
no pod goes unhealthy, nothing alerts about itself, the targets just go down and
any dashboard or alert built on Longhorn metrics goes blind.

The Longhorn **UI ingress is unaffected** — no policy selects `app: longhorn-ui`.

---

## 2. Option A — keep the hardening, add a scrape rule

The gap exists because upstream's policies assume Prometheus runs inside
`longhorn-system`. This adds back exactly the one path we need and nothing else:
one source namespace, one port.

Add as `infrastructure/safeqbit-local-hq/controllers/longhorn-scrape-policy.yaml`
and list it in that directory's `kustomization.yaml`.

```yaml
# Permits the one ingress path Longhorn 1.12's own policies leave out:
# Prometheus, in the monitoring namespace, scraping longhorn-manager metrics.
#
# Longhorn's NetworkPolicy/longhorn-manager admits same-namespace pods only
# (its `from` entries are podSelectors with no namespaceSelector), which is
# correct for its internal traffic and wrong for a Prometheus that lives
# elsewhere. NetworkPolicies are additive — this does not weaken Longhorn's
# policy, it unions one more source into what is allowed.
#
# Deliberately narrow: TCP 9500 only, and only from the Prometheus pods, not
# the whole monitoring namespace.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: longhorn-system
spec:
  podSelector:
    matchLabels:
      app: longhorn-manager
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9500
```

Note the `namespaceSelector` and `podSelector` are in the **same list element**,
so both must match (AND). Splitting them into two elements would admit the whole
monitoring namespace *and* every pod named prometheus anywhere (OR) — much wider
than intended.

`kubernetes.io/metadata.name: monitoring` is set automatically by Kubernetes on
every namespace; the `name: monitoring` label is also present but is ours to
break.

---

## 3. Option B — decline the hardening

```yaml
networkPolicies:
  restrictInternalTraffic: false
```

Preserves 1.11.3 behaviour exactly, creates no policies, pokes no holes. It
gives up the internal restriction wholesale to avoid writing one rule.

Per [`feedback-no-holes-in-flux-firewall`], the standing preference is not to
widen a controller's default policies — with an explicit carve-out for
"high-frequency, large-payload, or private-data cases". A 30-second scrape of a
pre-existing monitoring path is that carve-out, which is why Option A is the
recommendation rather than an assumption.

---

## 4. Preflight — all clean as of 2026-09-22

```bash
# V2 Data Engine must be off / no v2 volumes (v2 has no live upgrade)
kubectl get settings.longhorn.io -n longhorn-system v2-data-engine -o jsonpath='{.value}'   # false
kubectl get volumes.longhorn.io -n longhorn-system -o json \
  | jq '[.items[] | select(.spec.dataEngine=="v2")] | length'                                # 0

# No faulted volumes, no failed BackingImages
kubectl get volumes.longhorn.io -n longhorn-system -o json \
  | jq -r '.items[] | select(.status.robustness!="healthy") | .metadata.name'                # none of 25
kubectl get backingimages.longhorn.io -n longhorn-system                                     # none

# k8s floor is 1.25 in 1.12; we run 1.35
kubectl version -o json | jq -r .serverVersion.gitVersion
```

Engines roll themselves — `concurrentAutomaticEngineUpgradePerNodeLimit: 1` is
already in the HelmRelease `defaultSettings`. It defaults to `0` (disabled)
upstream, which is what left every volume on the old engine during the 1.11.3
upgrade; do not remove it.

---

## 5. What 1.12 actually changes for us

Mostly nothing. The headline features — fast volume cloning, storage sharding —
are **V2 Data Engine** only, and so is the single breaking change (legacy V2
linked-clone volumes can only be detached or deleted). We run V1.

Three CRDs are added: `enginefrontends`, `shardgroups`, `shards`.

Security: internal NetworkPolicies (section 1) and extended mTLS across all
instance-manager gRPC services.

---

## 6. Procedure

1. Longhorn system backup (UI → Setting → System Backup), and confirm recent
   Velero/CNPG restore points for anything stateful.
2. Merge the chart bump **with** the section 2 policy (or section 3 value) in
   the same change — the scrape gap should never exist, even briefly.
3. Watch the manager DaemonSet roll, then the engine live-upgrade (~5 min on
   1.11.3; zero downtime, volumes stay attached).
4. Verify with section 7.
5. Old-version instance-manager pods **legitimately linger** after an engine
   upgrade — engine processes stay in the IM pod they started in and only move
   when the workload pod restarts. Harmless. Do not force-delete them.
6. Once the old EngineImage CR hits `refCount: 0`, delete it:
   `kubectl delete engineimages.longhorn.io -n longhorn-system ei-<hash>`.
   Never while refCount > 0.

---

## 7. Verification — the scrape is the one that matters

```bash
# All volumes healthy, engines on the new image
kubectl get volumes.longhorn.io -n longhorn-system -o json \
  | jq -r '.items[] | select(.status.robustness!="healthy") | .metadata.name'
kubectl get engines.longhorn.io -n longhorn-system -o json \
  | jq -r '[.items[].spec.image] | unique[]'

# THE CHECK THIS DOC EXISTS FOR — must stay 3 targets, all up
kubectl exec -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 \
  -c config-reloader -- wget -qO- 'http://localhost:9090/api/v1/targets?state=active' \
  | jq -r '.data.activeTargets[] | select(.labels.job=="longhorn-backend")
           | .health + "  " + .scrapeUrl'
```

> The Prometheus container is **distroless** — no shell, no wget. Query through
> the `config-reloader` sidecar as above, not `-c prometheus`.

Also confirm the UI still loads (`longhorn.local.safeqbit.com`) and that CSI
still attaches — the simplest proof is that CNPG snapshots keep completing.

---

## Related

- [backup-strategy.md](backup-strategy.md) — Longhorn's place in the four layers
- [maintenance.md](maintenance.md) — general pod/volume ops
- Memory: `project-longhorn-1-12-networkpolicy-blocker`,
  `project-july-2026-upgrade-wave` (the 1.11.3 upgrade and the engine-limit fix),
  `project-prometheus-distroless-no-shell`
