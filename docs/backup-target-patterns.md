# Using MinIO as a Backup Target

An S3-compatible endpoint inside the cluster is an attractive backup target: no egress
cost, high throughput, and every backup tool already speaks S3. It also has a failure mode
that a remote target does not — **the thing you are backing up and the thing holding the
backups can die together**.

Start from that constraint.

## The 3-2-1 problem

MinIO running in the same cluster as the workloads it protects is one copy on one
infrastructure. It covers:

- Accidental deletion of a namespace or a table
- A bad migration or deploy
- Single-node and single-drive failures
- Restoring a cluster's workloads onto the same cluster

It does not cover:

- Loss of the cluster
- Loss of the datacentre / region
- An attacker with cluster-admin, who can reach the backup store too
- A control-plane failure that makes the backups unreachable exactly when needed

So: **in-cluster MinIO is the fast tier, not the only tier.** The workable pattern is a
local tenant for fast, frequent restores, plus site or bucket replication to a target on
different infrastructure with different credentials, for the disaster case.

```bash
# Bucket replication to an offsite target.
mc alias set offsite https://minio-dr.example.net OFFSITEKEY OFFSITESECRET
mc replicate add myminio/pgbackrest \
  --remote-bucket "https://OFFSITEKEY:OFFSITESECRET@minio-dr.example.net/pgbackrest" \
  --replicate "delete-marker,delete,existing-objects"
mc replicate status myminio/pgbackrest
```

Monitor the replication lag as an RPO metric, not as a queue depth — see
`../monitoring/minio-alerts.yaml`.

## Immutability

The single highest-value property of a backup target. A backup an attacker (or a script
with a bad loop) can delete is not a backup.

Three layers, all of which should be present:

### 1. Object Lock

Must be enabled **at bucket creation**; it cannot be added later.

```bash
mc mb --with-lock myminio/pgbackrest
mc retention set --default GOVERNANCE 30d myminio/pgbackrest
mc retention info --default myminio/pgbackrest
```

- `GOVERNANCE` — deletion requires the `s3:BypassGovernanceRetention` permission. A
  compromised backup agent cannot delete; a deliberate, privileged human can. This is the
  right default.
- `COMPLIANCE` — nothing and no one can delete before expiry, including root. Correct only
  when a regulator requires it. A mistyped retention period here is permanent and
  expensive.

### 2. An append-only policy for the backup agent

Covered in `access-policies.md`. The agent gets `PutObject` and `GetObject`, and an
explicit `Deny` on every delete verb. Expiry is then MinIO's job via lifecycle rules, not
the agent's.

This means turning **off** the backup tool's own retention pruning, or accepting that its
prune runs will log errors. Decide which, document it, and make sure the resulting log
noise does not train people to ignore backup job failures.

### 3. Versioning

```bash
mc version enable myminio/pgbackrest
```

Overwrites become new versions rather than destroying data. Required for replication and
Object Lock anyway.

Budget the capacity — see the versioning overhead discussion in `erasure-coding.md`, and
pair it with noncurrent-version expiry from `lifecycle-and-tiering.md`, set *longer* than
the lock retention or objects accumulate forever.

## Velero

Velero backs up Kubernetes object state to object storage, and (with the CSI plugin or
node-agent) volume data alongside it.

### BackupStorageLocation

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-credentials
  namespace: velero
type: Opaque
stringData:
  # Populated by External Secrets / Vault in real deployments.
  cloud: |
    [default]
    aws_access_key_id=VELERO_SERVICE_ACCOUNT_KEY
    aws_secret_access_key=VELERO_SERVICE_ACCOUNT_SECRET
---
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: minio
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: velero
    prefix: cluster-prod
  credential:
    name: minio-credentials
    key: cloud
  config:
    region: minio
    s3ForcePathStyle: "true"
    s3Url: https://minio-s3.minio-tenant.svc.cluster.local
    # Only if using the operator's self-signed CA and you have not mounted the bundle.
    insecureSkipTLSVerify: "false"
    caCert: <base64 CA bundle>
  # Velero's own pruning conflicts with an append-only policy. ReadOnly stops Velero
  # writing here at all, which is not what we want — instead, leave it writable and let
  # lifecycle rules handle expiry.
  accessMode: ReadWrite
```

Key details that cause real problems:

- **`s3ForcePathStyle: "true"` is mandatory** unless you have configured MinIO's bucket
  DNS. Virtual-host style addressing (`bucket.minio.svc`) does not resolve.
- **`region` must be set to something**, even though MinIO ignores it. The AWS SDK refuses
  to sign without one.
- **`prefix`** separates clusters sharing a bucket. Without it, two clusters backing up to
  the same bucket will read each other's backups and Velero's restore UI becomes a
  hazard.

### Schedule

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-full
  namespace: velero
spec:
  schedule: "0 2 * * *"
  useOwnerReferencesInBackup: false
  template:
    storageLocation: minio
    ttl: 720h                     # 30 days
    includedNamespaces:
      - "*"
    excludedNamespaces:
      - kube-system
      - velero
    excludedResources:
      - events
      - events.events.k8s.io
    snapshotVolumes: true
    defaultVolumesToFsBackup: false
    itemOperationTimeout: 4h
```

`ttl` tells Velero when to delete a backup. With an append-only policy the deletion fails
and the backup persists until the lifecycle rule expires it — which is the intended
behaviour, but it produces `BackupDeletionRequest` failures in the Velero logs. Set the
lifecycle expiry to match the intended retention and accept the noise, or grant Velero
delete rights and rely on Object Lock alone. Choose explicitly.

### The part people get wrong

Velero backs up *Kubernetes objects*. Volume data requires either CSI snapshots (which for
in-cluster MinIO means the snapshots live in the same cloud account) or file-system backup
via the node agent (which does copy bytes to the object store). A Velero backup with
`snapshotVolumes: true` and no CSI plugin installed produces a backup with no data in it,
and reports success. Verify what a backup actually contains:

```bash
velero backup describe daily-full-20260801020000 --details
velero backup logs daily-full-20260801020000
```

## pgBackRest

pgBackRest speaks S3 natively and is the better choice than Velero for Postgres, because
it produces a consistent, restorable database backup with WAL archiving rather than a
crash-consistent volume image.

```ini
[global]
repo1-type=s3
repo1-s3-endpoint=minio-s3.minio-tenant.svc.cluster.local
repo1-s3-bucket=pgbackrest
repo1-s3-region=minio
repo1-s3-uri-style=path
repo1-s3-key=PGBACKREST_SERVICE_ACCOUNT_KEY
repo1-s3-key-secret=PGBACKREST_SERVICE_ACCOUNT_SECRET
repo1-path=/prod-cluster

# Client-side encryption. The backup is unreadable to anyone with only bucket access —
# including a compromised MinIO. Keep this passphrase somewhere you can reach during a
# disaster that includes the cluster.
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=REPLACE_FROM_SECRET_MANAGER

repo1-retention-full=4
repo1-retention-diff=6
repo1-retention-archive-type=full
repo1-retention-archive=4

process-max=4
compress-type=zst
compress-level=6
start-fast=y
archive-async=y
spool-path=/var/spool/pgbackrest

[prod]
pg1-path=/var/lib/postgresql/data
pg1-port=5432
```

Points that matter:

- **`repo1-s3-uri-style=path`** — same path-style requirement as Velero.
- **`repo1-cipher-type`** — client-side encryption means the object store never sees
  plaintext. Store the passphrase outside the cluster; if it is only in a Kubernetes
  Secret in the cluster you are restoring, you have a circular dependency.
- **Retention vs. Object Lock.** pgBackRest expires old backups itself via
  `repo1-retention-full`. With an append-only policy those expirations fail. Either grant
  pgBackRest delete permission on its own prefix (and rely on Object Lock GOVERNANCE for
  immutability), or block deletes and let lifecycle rules do the expiry — in which case set
  retention generously so pgBackRest does not consider old backups gone while they still
  exist.
- **`archive-async=y`** with a spool path keeps WAL archiving from blocking Postgres when
  the object store is slow or briefly unavailable. Without it, a MinIO hiccup applies
  backpressure to the database. Monitor the spool directory size — if archiving stays
  broken, it fills the local disk and *then* Postgres stops.

Verify, always:

```bash
pgbackrest --stanza=prod check
pgbackrest --stanza=prod info
pgbackrest --stanza=prod --log-level-console=detail backup --type=full
```

## Restore rehearsals

Everything above is bookkeeping until a restore has been performed end to end. A rehearsal
that counts:

1. **On different infrastructure.** Restoring to the same cluster proves the backup exists,
   not that it is usable in a disaster.
2. **From the credentials and documentation available during an outage.** If the restore
   procedure needs a passphrase stored in the cluster being restored, it does not work.
3. **Timed.** Note the wall-clock time. That number, not the RTO in the policy document,
   is your actual recovery time.
4. **Validated at the application layer.** A Postgres that starts is not a Postgres with
   the right data. Run a query that would fail on a corrupt or stale restore.
5. **Scheduled.** Quarterly at minimum. Put it on a calendar with a named owner.

Write down what went wrong each time — the first rehearsal always finds something, and
that is the entire value.

## Monitoring the backup path itself

Alerting on backup jobs is not the same as alerting on backups. Both are needed:

- **Job succeeded** — the tool exited zero. Necessary, not sufficient.
- **Freshness** — the newest object under the prefix is younger than the RPO. This catches
  a job that succeeds while writing nothing.
- **Size sanity** — today's backup is within a sensible band of yesterday's. A backup that
  suddenly shrinks by 90% is the signal that matters most and the one nobody watches.
- **Replication lag to the offsite copy** — the real RPO for the disaster case.
- **Restore rehearsal recency** — an alert when the last verified restore is older than a
  quarter. Unusual, and worth it.

## Checklist

- [ ] Object Lock enabled at bucket creation, GOVERNANCE mode, retention set
- [ ] Backup agent has an append-only policy with explicit deletion `Deny`
- [ ] Versioning on, with noncurrent expiry longer than the lock retention
- [ ] Path-style addressing configured in every client
- [ ] Client-side encryption for database backups, passphrase stored outside the cluster
- [ ] Replication to infrastructure that does not share a failure domain with the source
- [ ] Freshness and size-sanity alerts, not just job-success alerts
- [ ] A dated record of the last successful restore rehearsal, and its wall-clock duration
