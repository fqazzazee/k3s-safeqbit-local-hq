# etcd B2 bucket — key rotation & bucket replacement

**Cluster:** safeqbit-local-hq
**Last reviewed:** 2026-10-23 (written before the first rotation; k3s
behaviour below checked against the v1.35.4+k3s1 source, the version the
nodes run)

Runbook for the B2 application key that the k3s servers use to upload etcd
snapshots ([backup-strategy.md → Layer 0](backup-strategy.md#layer-0---etcd-snapshots-to-b2-control-plane)).
Two procedures:

- **[A. Rotate the key](#a-rotate-the-key-same-bucket)**: new key, same bucket
  `k3s-safeqbit-etcd`. This is the routine case.
- **[B. Replace the bucket](#b-replace-the-bucket-new-bucket--new-key)**: new
  bucket and a new key, then retire the old bucket. Use it when you no longer
  trust what is in the bucket, or when you want bucket settings that can't be
  changed in place.

If you're doing this because the key may have leaked, read
[If the key leaked](#if-the-key-leaked) first.

No drain is needed for either procedure. A k3s restart leaves running pods
alone and costs a brief API blip on that node only. Work **one node at a
time** (etcd quorum), and stay clear of the Sunday 00:00 UTC scheduled
snapshot.

---

## Where the key lives

| Place | Notes |
|---|---|
| `/etc/rancher/k3s/config.yaml` on k3s-server-01/02/03 | `etcd-s3-access-key` / `etcd-s3-secret-key`, root-only 0600. The only place k3s reads the key from. |
| Vaultwarden | The copy for rebuilds and restores ([node-bootstrap.md](node-bootstrap.md)). |
| Proxmox VM backups of the server VMs | Stale copies of `config.yaml`. Revoking the old key makes them harmless. |

It is **not** in Git or in any cluster Secret (no `etcd-s3-config-secret`).
The `k3s.io/node-args` node annotation shows both flags redacted as
`********`. `ETCDSnapshotFile` records carry only bucket, endpoint, region and
prefix. All of this was checked on 2026-10-23.

---

## How k3s uses the key (what the checks below rely on)

1. **The running server reads `config.yaml` only at start.** Scheduled
   snapshots use whatever key was in the file at the last k3s restart. A
   file edit without a restart changes nothing for the Sunday run.
2. **`sudo k3s etcd-snapshot save` uses the key in the *file*, not the one
   the server is running with.** The CLI parses `config.yaml` and sends its
   S3 settings with the request, and the server uses them for that one
   snapshot. That makes it a good pre-restart test of the edited file, but
   it does **not** prove the scheduled snapshots will work.
3. **Proof that the running server's key works:** on start, k3s reconciles
   its snapshot records. That reconcile opens an S3 client (`BucketExists`
   against the configured bucket) and, only if that succeeds, stamps the
   Node annotation `etcd.k3s.cattle.io/s3-snapshots-timestamp`. If the
   timestamp is **later than the restart**, the running server reached the
   bucket with its key. If the key or bucket is wrong, the timestamp stays
   old and `journalctl -u k3s` shows `failed to initialize S3 client`.
4. **Records follow the configured bucket.** The reconcile lists the bucket
   currently configured and deletes every `ETCDSnapshotFile` record with label
   `etcd.k3s.cattle.io/snapshot-storage-node=s3` that isn't in it, for all
   nodes and not only this one. Deleting a record **never deletes the
   object**. If the S3 listing fails, records are left alone. So a broken key
   does **not** make `kubectl get etcdsnapshotfiles` look broken: check (3),
   not the record list.
5. **S3 retention is 8 snapshots across all three nodes**, not 8 per node.
   k3s prunes `snapshots/etcd-snapshot*` as one pool. The bucket therefore
   holds the newest 8 scheduled snapshots, which is under 3 weeks of
   history. On-demand snapshots (`snapshots/on-demand*`) are a separate pool
   of 8, pruned only when another on-demand save runs. Node-local retention
   is 8 per node under `/var/lib/rancher/k3s/server/db/snapshots/`.

Check commands used below (run from the workstation):

```bash
NODE=k3s-server-0X

# (3) server-side proof — must be AFTER the k3s restart time
kubectl get node $NODE \
  -o jsonpath='{.metadata.annotations.etcd\.k3s\.cattle\.io/s3-snapshots-timestamp}{"\n"}'

# B2 records, newest last
kubectl get etcdsnapshotfiles --sort-by=.status.creationTime \
  -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,BUCKET:.spec.s3.bucket,CREATED:.status.creationTime,READY:.status.readyToUse \
  | grep -v '<none>'
```

---

## A. Rotate the key (same bucket)

### A1. Mint the new key (B2 console)

1. **Application Keys**: look at the current key's row and note its bucket,
   capabilities, file-name prefix and whether it can list all bucket names.
   The console shows these, but never the secret.
2. **Add a New Application Key** with the same settings:
   - name: `k3s-etcd-YYYYMM`
   - access: bucket `k3s-safeqbit-etcd` **only**, which keeps the
     blast-radius separation from the Velero key
   - type of access: Read and Write (k3s uploads, lists and prunes)
   - prefix / list-all-bucket-names: the same as the old key
3. Save the keyID and applicationKey to Vaultwarden **now**, because B2 shows
   the secret once. Keep the old key's entry for the moment.

**Do not delete the old key yet.** Until A3 the running servers still use it.

### A2. Roll it onto each node — one at a time

Use any order, for example 03 → 02 → 01. On the node:

```bash
sudo -e /etc/rancher/k3s/config.yaml   # replace etcd-s3-access-key / etcd-s3-secret-key
                                       # (an editor, not sed: keeps the key out of shell history)
sudo stat -c '%a %U:%G' /etc/rancher/k3s/config.yaml   # expect: 600 root:root

# Pre-restart test of the FILE (see "How k3s uses the key" 2).
sudo k3s etcd-snapshot save
#   success → an s3://k3s-safeqbit-etcd/snapshots/on-demand-… upload is logged
#   failure → fix the file now; the server is still on the old key, nothing is broken
#   ("Unknown flag --tls-san" warning is harmless)

sudo systemctl restart k3s
```

From the workstation, before touching the next node:

```bash
kubectl get nodes                          # all 3 Ready
kubectl get --raw='/healthz/etcd'; echo    # ok
# + the s3-snapshots-timestamp check above → later than the restart
```

Restarting k3s-server-01 also clears its stale kubelet memory capacity
([node-bootstrap.md](node-bootstrap.md)). Expect `kubectl top nodes` to change
for that node.

### A3. Revoke the old key

Only after all three nodes pass the timestamp check:

1. B2 console → Application Keys → delete the old key.
2. Vaultwarden: delete the old key's entry, or mark it revoked.
3. After the next Sunday 00:00 UTC run: three new `etcd-snapshot-…` records,
   one per node, all `READY true` (or `/cluster etcd` in Slack).
4. Optional: drop the test snapshots from A2. Run
   `sudo k3s etcd-snapshot delete <snapshotName>` on a node; this removes the
   object from B2 as well. They're about 65 MB each, and the on-demand pool
   prunes itself at 8 anyway.
5. Add a row to the [Rotation log](#rotation-log).

---

## B. Replace the bucket (new bucket + new key)

**Space:** the B2 free tier is 10 GB (decimal) per *account*, shared with
the Velero bucket. The old bucket keeps its live snapshots plus a 30-day
hidden-version tail until you delete it, and the new bucket fills at about
200 MB per Sunday (3 × ~65 MB) up to the 8-snapshot cap. Check total usage in
the console before starting.

### B1. Create the bucket and its key

1. **Name:** B2 bucket names are unique across *all* B2 accounts. Pick a new
   name, for example `k3s-safeqbit-etcd-2`, and don't plan on reusing
   `k3s-safeqbit-etcd`.
2. **Settings:** Private. Match the old bucket's default encryption and
   Object Lock settings, or decide them now if changing them is why you're
   doing this. Region is fixed per account (`us-east-005`), so the endpoint
   doesn't change.
3. **Lifecycle:** "Keep prior versions for this number of days: 30". This is
   the undelete window from Layer 0: a deletion made with the node-side key
   can be undone from the console for a month.
4. **Key:** as in [A1](#a1-mint-the-new-key-b2-console), but scoped to the
   **new** bucket.

### B2. Roll it onto each node — one at a time

The same as [A2](#a2-roll-it-onto-each-node--one-at-a-time), except that you
also change one more line in `config.yaml`:

```yaml
etcd-s3-bucket: <new-bucket>
etcd-s3-access-key: <new keyID>
etcd-s3-secret-key: <new applicationKey>
```

The pre-restart `save` should log an upload to `s3://<new-bucket>/…`.

**Expect records to flip while you roll.** Each node's reconcile drops the
records that aren't in *its* bucket ("How k3s uses the key" 4). A switched
node removes the old-bucket records, and a node not yet switched recreates
them on its next reconcile. This touches records only, never objects, and
stops once all three nodes are switched. `/cluster etcd` may show odd counts
in the meantime.

### B3. Verify

After all three nodes:

- the timestamp check passes on every node
- the record listing shows **only** `<new-bucket>` in the BUCKET column. The
  old bucket's records are gone, and its objects are still in the old bucket.
- after the next Sunday run: three new scheduled records in the new bucket

### B4. Revoke the old key, keep the old bucket for now

1. Delete the old key (as in [A3](#a3-revoke-the-old-key)).
2. **Keep the old bucket** until the new one holds a full history: 8
   scheduled snapshots, which takes 3 Sundays. To restore from the old bucket
   in the meantime, mint a temporary **read-only** key for it in the console
   and pass it to the restore command in
   [node-bootstrap.md → Path A](node-bootstrap.md) together with the old
   bucket name.
3. If you're replacing the bucket because of a leak, treat the old bucket's
   snapshots as untrusted for restores. The node-local copies (one directory
   per server, under `/var/lib/rancher/k3s/server/db/snapshots/`) and the new
   bucket are the trusted copies.

### B5. Delete the old bucket

B2 won't delete a bucket that still holds files, and hidden prior versions
count. To empty it, set the old bucket's lifecycle to a custom rule:

```json
[{"fileNamePrefix": "", "daysFromUploadingToHiding": 1, "daysFromHidingToDeleting": 1}]
```

Lifecycle runs once a day, so the bucket is empty after about 2 days. Then
use **Delete Bucket** in the console. If the old bucket has Object Lock with
retention, locked versions can't be removed until their retention expires.

### B6. Docs sweep

```bash
grep -rn 'k3s-safeqbit-etcd' workbook/ infrastructure/
```

Update the current-state references: [node-bootstrap.md](node-bootstrap.md)
(the config block and the Path A restore command),
[backup-strategy.md](backup-strategy.md) Layer 0, [maintenance.md](maintenance.md)
"etcd Snapshots", [restore-drills.md](restore-drills.md), and the
Slack-bot wiki ConfigMap (`infrastructure/safeqbit-local-hq/configs/cluster-slack-bot-wiki.yaml`,
which live-syncs and needs no restart). Leave dated history alone: the
improvement-plan entry and the backup-strategy changelog rows. Then add a row
to the [Rotation log](#rotation-log).

---

## If the key leaked

- **Treat the snapshots as read.** Secrets encryption at rest is **not**
  enabled on this cluster (no `--secrets-encryption` in the node args,
  checked 2026-10-23). Every snapshot therefore holds every Secret in
  plaintext, including the **sealed-secrets private key**. Rotating the B2
  key stops future access but doesn't undo a download, and there is no
  download log for the bucket to rule one out.
- **Treat the snapshots as possibly altered.** The key has write access, so
  snapshots could have been overwritten (a planted snapshot would be
  restored as cluster state) or deleted (the 30-day prior-version tail in
  the console covers deletions). Use **procedure B**, not A.
- **Do it now, not on a maintenance window:** mint the new key, then
  **revoke the old one first**. Losing a few hours of uploads is
  better than leaving a leaked key live while you roll the nodes.
- **Afterwards:** rotate the sealed-secrets key and reseal
  ([maintenance.md → Sealed Secrets](maintenance.md#sealed-secrets)), and
  rotate the credentials of anything that matters (B2 Velero key, Authentik,
  Vaultwarden admin, Cloudflare/Pangolin tokens, CNPG passwords). That is
  separate work and is not covered here.

---

## Rotation log

| Date | Procedure | Key name / bucket | Notes |
|---|---|---|---|
| | | | |
