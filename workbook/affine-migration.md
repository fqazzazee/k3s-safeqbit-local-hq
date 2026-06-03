# AFFiNE Migration Runbook

**Migrated from:** Docker host (Portainer stack) → safeqbit-local-hq k3s cluster  
**Date:** 2026-05-15  
**Namespace:** `affine`  
**Hostname:** `affine.local.safeqbit.com`

---

## Architecture changes from Docker Compose

| Docker Compose | k3s |
|---|---|
| `pgvector/pgvector:pg16` container | CNPG Cluster CR (`affine-cnpg`, instances: 1) |
| `redis:7-alpine` (no auth, no persistence) | Redis Deployment + ClusterIP, with password |
| `affine_migration` service (`restart: "no"`) | `affine-migration` initContainer (idempotent) |
| macvlan static IP on LAN | nginx Ingress + cert-manager TLS |
| Docker volumes on host path | Longhorn PVCs (`affine-storage` 10Gi, `affine-config` 1Gi) |

**pgvector note:** The CNPG cluster uses standard PostgreSQL 16 (no pgvector).
AFFiNE's migration script skips the vector extension when `AFFINE_INDEXER_ENABLED=false`.
If you ever enable the indexer, you'll need to rebuild with a pgvector image - see the
comment in `03-cnpg-cluster.yaml`.

---

## Prerequisites

Collect from the running Portainer stack on the Docker host:

- The Docker host IP / SSH access
- Paths to `UPLOAD_LOCATION` and `CONFIG_LOCATION` from the Portainer stack `.env`
- A strong random string to use as the Redis password (generate with:
  `openssl rand -base64 32`)

---

## Step 1 - Seal credentials

Generate a Redis password and seal it:

```bash
kubectl create secret generic affine-credentials \
  --namespace affine \
  --dry-run=client \
  --from-literal=redis-password='<generated-password>' \
  -o yaml | \
/tmp/kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  -o yaml > apps/safeqbit-local-hq/affine/02-sealed-secrets.yaml
```

---

## Step 2 - Commit and push manifests

```bash
git add apps/safeqbit-local-hq/affine/ \
        apps/safeqbit-local-hq/kustomization.yaml
git commit -m "apps: add AFFiNE migration to k3s"
git push origin main

# Trigger Flux immediately
kubectl annotate gitrepository flux-system -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite

# Wait for CNPG to initialise
until kubectl get cluster affine-cnpg -n affine \
  -o jsonpath='{.status.readyInstances}' 2>/dev/null | grep -q "1"; do sleep 5; done
echo "CNPG ready"

# Wait for AFFiNE to pass readiness (boots against empty DB - expected)
kubectl rollout status deploy/affine -n affine --timeout=5m
```

At this point AFFiNE has booted against an empty database and the migration
initContainer has created the schema. This is expected - we wipe and restore next.

---

## Step 3 - Scale to zero before data restore

```bash
kubectl scale deploy/affine -n affine --replicas=0
kubectl wait --for=delete pod -l app.kubernetes.io/name=affine \
  -n affine --timeout=60s
```

---

## Step 4 - Dump Postgres from Docker host

SSH to the Docker host and dump the database:

```bash
# On the Docker host
docker exec affine_postgres \
  pg_dump -U affine --clean --if-exists affine > /tmp/affine.sql
```

Copy the dump to your local machine:

```bash
scp user@<docker-host-ip>:/tmp/affine.sql ./affine.sql
```

---

## Step 5 - Restore Postgres into CNPG

Drop the empty schema that the migration initContainer created, then restore:

```bash
# Drop and restore in one pipeline
(
  echo "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
  cat affine.sql
) | kubectl exec -i affine-cnpg-1 -n affine -- \
  psql -U affine affine

# Verify row counts look sane
kubectl exec -i affine-cnpg-1 -n affine -- \
  psql -U affine affine -c "\dt" | head -30
```

---

## Step 6 - Copy storage and config volumes

Sync files from the Docker host to your local machine first:

```bash
# Adjust paths to match UPLOAD_LOCATION and CONFIG_LOCATION from the .env
rsync -av user@<docker-host-ip>:/path/to/UPLOAD_LOCATION/ ./affine-storage/
rsync -av user@<docker-host-ip>:/path/to/CONFIG_LOCATION/ ./affine-config/
```

Spin up a restore pod that mounts both PVCs:

```bash
kubectl run affine-restore -n affine --restart=Never \
  --image=alpine \
  --overrides='{
    "spec": {
      "volumes": [
        {"name":"storage","persistentVolumeClaim":{"claimName":"affine-storage"}},
        {"name":"config","persistentVolumeClaim":{"claimName":"affine-config"}}
      ],
      "containers": [{
        "name": "c",
        "image": "alpine",
        "command": ["sleep","3600"],
        "volumeMounts": [
          {"name":"storage","mountPath":"/storage"},
          {"name":"config","mountPath":"/config"}
        ]
      }]
    }
  }'

kubectl wait --for=condition=Ready pod/affine-restore -n affine --timeout=60s

# Copy files into the PVCs
kubectl cp ./affine-storage/. affine/affine-restore:/storage/
kubectl cp ./affine-config/.  affine/affine-restore:/config/

# Verify private key is present
kubectl exec -n affine affine-restore -- ls -la /config/

kubectl delete pod affine-restore -n affine
```

---

## Step 7 - Scale up and verify

```bash
kubectl scale deploy/affine -n affine --replicas=1
kubectl rollout status deploy/affine -n affine --timeout=5m

# Watch logs for startup errors
kubectl logs -n affine -l app.kubernetes.io/name=affine -f --tail=50
```

Open `https://affine.local.safeqbit.com` and confirm:

- [ ] Login works
- [ ] Existing workspaces are visible
- [ ] Notes and pages load correctly
- [ ] Attached images/files open

---

## Step 8 - Decommission Docker stack

Once verified, stop the Portainer stack on the Docker host:

```bash
# On the Docker host
docker compose -p affine down
```

Remove the stack from Portainer and delete the old volumes at your convenience.

---

## Expanding storage later

If `affine-storage` fills up, expand it online - no pod restart needed:

```bash
kubectl patch pvc affine-storage -n affine \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Confirm Longhorn has resized
kubectl get pvc affine-storage -n affine
```
