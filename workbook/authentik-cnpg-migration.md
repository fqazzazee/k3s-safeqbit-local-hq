# Authentik: Migrating from StatefulSet Postgres to CloudNativePG

**Date:** 2026-05-14  
**Cluster:** safeqbit-local-hq

---

## What Changed and Why

Authentik's database was previously a hand-rolled Postgres 16 StatefulSet — a single pod mounting a Longhorn PVC. It worked but offered no automatic failover, no application-consistent backups (Velero snapshotted the PVC without Postgres being aware), and no built-in monitoring integration.

CloudNativePG (CNPG) is a Kubernetes operator that manages Postgres clusters as first-class Kubernetes objects. By moving to CNPG we get:

- A `Cluster` custom resource that declaratively describes the Postgres cluster
- Automatic pod recreation with correct sequencing if the pod dies
- Services for read-write (`-rw`), read-only (`-ro`), and any replica (`-r`) that the operator keeps up to date as primaries change
- Auto-generated app credentials stored in a Kubernetes secret
- Application-consistent backups via `ScheduledBackup` (Postgres checkpoints before the snapshot, so the snapshot is always clean)
- A built-in `PodMonitor` for Prometheus scraping

---

## Architecture: Old vs New

### Old Setup

```
┌──────────────────────────────────────────┐
│               authentik namespace         │
│                                           │
│  ┌───────────────┐    ┌────────────────┐  │
│  │  StatefulSet  │    │  HelmRelease   │  │
│  │  authentik-   │◄───│  (Authentik    │  │
│  │  postgres-0   │    │   server +     │  │
│  │  (postgres:16)│    │   worker)      │  │
│  └───────┬───────┘    └────────────────┘  │
│          │ mounts                         │
│  ┌───────▼───────┐                        │
│  │  Longhorn PVC │  ← Velero snapshots    │
│  │  (10Gi)       │    this PVC directly,  │
│  └───────────────┘    without Postgres    │
│                       awareness           │
└──────────────────────────────────────────┘
```

**Problems with this approach:**
- Velero took a filesystem-level snapshot of the PVC while Postgres was running. Postgres keeps pages in shared_buffers (memory) that may not have been flushed to disk. A restore from such a snapshot could give a corrupted or inconsistent database.
- No automated failover. If the pod died, Kubernetes restarted it, but any in-flight transactions were lost.
- Credentials (`POSTGRES_PASSWORD`) were manually managed in the sealed secret.

### New Setup

```
┌──────────────────────────────────────────────────┐
│                 authentik namespace               │
│                                                   │
│  ┌─────────────────────────────┐                  │
│  │   CNPG Cluster CR           │                  │
│  │   (authentik-cnpg)          │                  │
│  │                             │                  │
│  │  ┌─────────────────────┐   │  ┌─────────────┐ │
│  │  │  authentik-cnpg-1   │   │  │HelmRelease  │ │
│  │  │  (Postgres 16 pod)  │◄──┼──│(server +    │ │
│  │  └──────────┬──────────┘   │  │ worker)     │ │
│  │             │ CNPG-managed │  └─────────────┘ │
│  │  ┌──────────▼──────────┐   │                  │
│  │  │  Longhorn PVC       │   │                  │
│  │  │  (auto-created)     │   │                  │
│  │  └─────────────────────┘   │                  │
│  │                             │                  │
│  │  Services (auto-created):   │                  │
│  │  • authentik-cnpg-rw  ──────┼──► primary       │
│  │  • authentik-cnpg-ro        │                  │
│  │  • authentik-cnpg-r         │                  │
│  │                             │                  │
│  │  Secrets (auto-created):    │                  │
│  │  • authentik-cnpg-app ──────┼──► HelmRelease   │
│  └─────────────────────────────┘                  │
│                                                   │
│  ┌─────────────────────────────┐                  │
│  │  ScheduledBackup CR         │                  │
│  │  (daily 01:00 UTC)          │                  │
│  │  method: volumeSnapshot     │                  │
│  └─────────────────────────────┘                  │
└──────────────────────────────────────────────────┘

       ▲ CNPG operator pod lives in cnpg-system
         but watches ALL namespaces
```

---

## How CNPG Works

### The Operator Model

The CNPG operator runs in `cnpg-system` and watches all namespaces for `Cluster` CRs. When you create a `Cluster` CR, the operator:

1. Creates one Postgres pod per `spec.instances`
2. Creates a Longhorn PVC for each pod (naming: `<cluster-name>-<instance-number>`)
3. Runs `initdb` to initialize the database with the configured owner and database name
4. Creates three Services pointing at the current primary (`-rw`), hot standbys (`-ro`), and any instance (`-r`)
5. Creates secrets for the app user (`<cluster-name>-app`) and handles the TLS certificates for internal replication

### The Cluster CR (our config)

```yaml
spec:
  instances: 1              # number of Postgres pods (1 = single primary, no replicas)
  bootstrap:
    initdb:
      database: authentik   # creates this DB on first boot
      owner: authentik       # creates this user as DB owner; CNPG generates its password
  storage:
    storageClass: longhorn
    size: 10Gi
  backup:
    volumeSnapshot:
      className: longhorn-velero   # which VolumeSnapshotClass to use
      snapshotOwnerReference: backup  # VolumeSnapshot lifetime is tied to its Backup object
```

### Auto-Generated Secret: `authentik-cnpg-app`

CNPG creates this secret automatically with a randomly generated password. It contains 11 keys:

| Key | Example value |
|-----|---------------|
| `username` | `authentik` |
| `password` | `<generated>` |
| `host` | `authentik-cnpg-rw` |
| `port` | `5432` |
| `dbname` | `authentik` |
| `uri` | `postgresql://authentik:<pwd>@authentik-cnpg-rw.authentik:5432/authentik` |
| `pgpass` | Postgres `.pgpass` format |
| `fqdn-uri` | URI with fully-qualified hostname |
| `jdbc-uri` | JDBC connection string |
| `fqdn-jdbc-uri` | JDBC with FQDN |

The Authentik HelmRelease reads `AUTHENTIK_POSTGRESQL__PASSWORD` directly from this secret, so no manual password management is needed.

---

## How Backups Work

### Old: Velero CSI Snapshot (DB-unaware)

```
Velero schedule fires
    │
    ▼
Velero tells Longhorn: "snapshot this PVC"
    │
    ▼
Longhorn takes a crash-consistent snapshot
    │   ← Postgres is still running, dirty pages may be in RAM
    ▼
Snapshot is offloaded to B2 via data mover
```

**Risk:** A restore from this snapshot may require Postgres crash recovery. Usually succeeds, but is not guaranteed to be logically consistent (a transaction that was mid-commit could be in a partial state on disk).

### New: CNPG ScheduledBackup (DB-aware)

```
ScheduledBackup fires (daily 01:00 UTC)
    │
    ▼
CNPG tells Postgres: CHECKPOINT (flush all dirty pages to disk)
    │
    ▼
Postgres confirms checkpoint complete
    │
    ▼
CNPG tells Longhorn: "snapshot this PVC" (via VolumeSnapshot API)
    │   ← Postgres data on disk is now fully consistent
    ▼
VolumeSnapshot created (using longhorn-velero class)
    │
    ▼
Backup object created, tied to VolumeSnapshot lifecycle
```

**What you get:** A snapshot that represents Postgres at a known-clean checkpoint. Restoring from it needs no crash recovery — the database is in a consistent state.

**Velero still runs biweekly** and covers the rest of the `authentik` namespace (media PVC, templates PVC, the sealed secret). It also snapshots the CNPG PVC, but CNPG's daily backup is what you'd use to restore the database specifically.

### Backup Retention

| Schedule | Cadence | Retention | Covers |
|----------|---------|-----------|--------|
| `authentik-cnpg-backup` | Daily 01:00 UTC | Until Backup object is deleted (no TTL set) | DB volumes only, DB-aware |
| `authentik-frequent` | 1st & 15th at 03:00 UTC | 6 months | Full `authentik` namespace |
| `weekly-everything` | 1st & 15th at 03:00 UTC | 3 months | `authentik` + `vaultwarden` namespaces |

---

## Migration Runbook

### Pre-requisites

- CNPG operator deployed and healthy (`kubectl get pods -n cnpg-system`)
- New manifests pushed to Git and Flux reconciled
- Velero schedules updated in Git

### Step 1 — Suspend the HelmRelease

Flux reconciled the updated HelmRelease almost immediately after the push, which caused Authentik to run Django migrations on the empty CNPG database before we could stop it. **For future migrations: suspend the HelmRelease before pushing**, not after.

```bash
kubectl patch helmrelease authentik -n authentik \
  --type=merge -p '{"spec":{"suspend":true}}'
```

This patches the `HelmRelease` object's `spec.suspend` field to `true`. Flux's helm-controller respects this field and stops reconciling the release until it's set back to `false`. Equivalent to `flux suspend helmrelease -n authentik authentik` when the `flux` CLI is available.

### Step 2 — Wait for the CNPG Cluster to be ready

```bash
kubectl wait cluster -n authentik authentik-cnpg \
  --for=condition=Ready --timeout=180s
```

`kubectl wait` polls the resource until the named condition is true or the timeout expires. `condition=Ready` checks the `Ready` status condition that CNPG sets on its `Cluster` objects. At this point the `authentik-cnpg-app` secret exists and the `authentik-cnpg-rw` service is live.

### Step 3 — Wipe the contaminated CNPG database

Because Authentik ran migrations on the fresh empty database before we suspended, we had to wipe it clean:

```bash
CNPG_PASS=$(kubectl get secret -n authentik authentik-cnpg-app \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl exec -i -n authentik authentik-cnpg-1 -- \
  sh -c "PGPASSWORD='${CNPG_PASS}' psql -U authentik \
         -h authentik-cnpg-rw authentik" <<'EOF'
DROP SCHEMA IF EXISTS public CASCADE;
CREATE SCHEMA public;
DROP SCHEMA IF EXISTS template CASCADE;
CREATE SCHEMA template;
EOF
```

**Breaking this down:**

- `kubectl get secret ... -o jsonpath='{.data.password}'` — extracts a single key from a Kubernetes secret in its base64-encoded form
- `| base64 -d` — decodes it to plaintext
- `kubectl exec -i -n authentik authentik-cnpg-1 --` — runs a command inside the CNPG pod; `-i` keeps stdin open so we can pipe the heredoc SQL into it
- The `sh -c "PGPASSWORD=... psql ..."` pattern sets the password as an environment variable just for that one command, rather than storing it anywhere
- `<<'EOF' ... EOF` — a heredoc that feeds multiple SQL statements to psql's stdin; the single-quotes around `EOF` prevent shell variable expansion inside the block
- `DROP SCHEMA ... CASCADE` — drops the schema and everything in it (192 tables, indexes, functions, etc.)

### Step 4 — Dump and restore

```bash
CNPG_PASS=$(kubectl get secret -n authentik authentik-cnpg-app \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl exec -n authentik authentik-postgres-0 -- \
  sh -c 'PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
         -U authentik -h localhost --clean --if-exists authentik' \
| kubectl exec -i -n authentik authentik-cnpg-1 -- \
  sh -c "PGPASSWORD='${CNPG_PASS}' psql \
         -U authentik -h authentik-cnpg-rw authentik"
```

**Breaking this down:**

- The entire command is two `kubectl exec` calls connected by a Unix pipe (`|`). The left side runs inside the old StatefulSet pod; the right side runs inside the CNPG pod. Data flows directly between them through your local shell — no intermediate file needed.
- `PGPASSWORD=$POSTGRES_PASSWORD` — the old StatefulSet pod already has `POSTGRES_PASSWORD` set as an environment variable (from the SealedSecret). We reference it as `$POSTGRES_PASSWORD` inside the exec so the shell inside the container expands it (not the local shell).
- `pg_dump --clean --if-exists` — produces SQL that includes `DROP ... IF EXISTS` before each `CREATE`. This is what makes re-running the restore safe: if an object already exists it is dropped first, then recreated. Without `--clean`, restoring into a database that already has the schema produces hundreds of "already exists" errors and may leave partial data.
- `-h localhost` — connects to the Postgres server via TCP on localhost inside the pod (rather than the Unix socket, which may require being the `postgres` OS user).
- The right-hand `psql -h authentik-cnpg-rw` — connects to the CNPG primary Service from inside the CNPG pod. You could also connect from any other pod in the namespace.

### Step 5 — Verify row counts match

```bash
# Check CNPG
kubectl exec -i -n authentik authentik-cnpg-1 -- \
  sh -c "PGPASSWORD='${CNPG_PASS}' psql \
         -U authentik -h authentik-cnpg-rw authentik" <<'EOF'
SELECT 'users' AS table, COUNT(*) FROM authentik_core_user
UNION ALL SELECT 'flows',      COUNT(*) FROM authentik_flows_flow
UNION ALL SELECT 'providers',  COUNT(*) FROM authentik_core_provider
UNION ALL SELECT 'migrations', COUNT(*) FROM django_migrations;
EOF
```

Run the same query against the old pod and confirm the numbers match before proceeding.

### Step 6 — Resume the HelmRelease

```bash
kubectl patch helmrelease authentik -n authentik \
  --type=merge -p '{"spec":{"suspend":false}}'
```

Flux reconciles the HelmRelease, deploys Authentik pointing at `authentik-cnpg-rw`, and reads the password from `authentik-cnpg-app`. The `Ready` condition on the HelmRelease should show `Helm upgrade succeeded`.

### Step 7 — Health-check Authentik

```bash
kubectl exec -n authentik <server-pod> -- \
  curl -sf http://localhost:9000/-/health/ready/ && echo "OK"
```

`/-/health/ready/` is Authentik's readiness endpoint. It returns HTTP 200 only when the server has a working database connection and has finished startup. Using `-sf` with `curl` makes it fail silently on HTTP errors and suppresses progress output — the `&& echo "OK"` only prints if curl exits 0.

### Step 8 — Remove old resources

Because the apps Flux Kustomization has `prune: false`, removing files from Git does not delete the corresponding cluster resources. Manual cleanup is required:

```bash
kubectl delete statefulset -n authentik authentik-postgres
kubectl delete service     -n authentik authentik-postgres
kubectl delete pvc         -n authentik authentik-postgres-data
```

Order matters here: deleting the StatefulSet first ensures the pod is gone before we delete the PVC it was mounting (deleting a PVC while a pod still mounts it puts it in a stuck `Terminating` state).

---

## Sealed Secret Cleanup

The original `authentik-credentials` SealedSecret had three keys:

| Key | Used by | Status after migration |
|-----|---------|----------------------|
| `AUTHENTIK_SECRET_KEY` | Authentik HelmRelease | Still needed |
| `AUTHENTIK_POSTGRESQL__PASSWORD` | HelmRelease (old) | Removed — CNPG app secret used instead |
| `POSTGRES_PASSWORD` | StatefulSet (old) | Removed — StatefulSet is gone |

To re-seal with only the surviving key, we need the `kubeseal` binary and the controller's public certificate:

```bash
# Fetch the public cert from the running controller
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o jsonpath='{.items[0].data.tls\.crt}' | base64 -d > /tmp/sealed-secrets.crt

# Read the plaintext value from the live (decrypted) secret
SECRET_KEY=$(kubectl get secret -n authentik authentik-credentials \
  -o jsonpath='{.data.AUTHENTIK_SECRET_KEY}' | base64 -d)

# Build a plain Secret manifest and pipe it to kubeseal
kubectl create secret generic authentik-credentials \
  --namespace authentik \
  --from-literal=AUTHENTIK_SECRET_KEY="${SECRET_KEY}" \
  --dry-run=client -o yaml \
| kubeseal \
    --cert /tmp/sealed-secrets.crt \
    --format yaml > apps/safeqbit-local-hq/authentik/02-sealed-secrets.yaml
```

**Why `--dry-run=client -o yaml`?** — `kubectl create secret ... --dry-run=client` generates the Secret manifest without applying it to the cluster. Piping the output to `kubeseal` encrypts it using the controller's public key. The result is a `SealedSecret` that can be committed to Git — the controller is the only thing in the cluster that can decrypt it.

**Why fetch the cert separately instead of letting kubeseal call the API?** — The controller at `kube-system/sealed-secrets-controller` exposes its cert via its API, but fetching it directly from the `tls.crt` Secret label is more reliable when the cert endpoint has RBAC restrictions. Both approaches produce the same cert.

---

## Key Files After Migration

```
apps/safeqbit-local-hq/authentik/
├── 01-namespace.yaml              # Namespace definition (unchanged)
├── 02-sealed-secrets.yaml         # SealedSecret — now only AUTHENTIK_SECRET_KEY
├── 03-cnpg-cluster.yaml           # CNPG Cluster CR (replaces 03+04+05)
├── 04-cnpg-scheduled-backup.yaml  # Daily VolumeSnapshot backup
├── 06-shared-pvcs.yaml            # Media and templates PVCs (unchanged)
├── 07-helmrelease.yaml            # Authentik chart — host: authentik-cnpg-rw
└── 08-ingress.yaml                # Ingress (unchanged)

infrastructure/safeqbit-local-hq/configs/
├── velero-schedule-authentik.yaml # Biweekly (1st & 15th), 6-month retention
└── velero-schedule-weekly.yaml    # Biweekly (1st & 15th), 3-month retention
```

---

## Lessons Learned

**Suspend the HelmRelease before pushing DB-breaking changes.** Flux reconciles within seconds of a push. If the new HelmRelease points at a database that doesn't have data yet, the application will start, run migrations, and seed default data into the empty database — contaminating the restore target. The fix in this case was to wipe the schemas and redo the dump with `--clean --if-exists`, but the safer pattern is:

```
suspend HelmRelease → push → wait for CNPG → dump/restore → resume HelmRelease
```

**`prune: false` means manual cleanup.** The apps Kustomization has `prune: false`, which means Flux never deletes cluster resources when their manifests are removed from Git. This is a deliberate safety choice for a homelab, but it means old StatefulSets, Services, and PVCs linger until you delete them manually.

**CNPG does not create a superuser secret by default.** Only `<cluster-name>-app` is created unless you add `superuserSecret` to the Cluster spec. Use the app user credentials for routine database access.
