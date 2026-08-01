# Lifecycle Management and Tiering

Object storage grows until something tells it to stop. Lifecycle rules are that something,
and they should exist before the first object is written — retrofitting expiry onto a
bucket with 400 million objects is a multi-day scan.

## The three levers

1. **Expiration** — delete objects (or versions) after N days.
2. **Transition** — move objects to a cheaper remote tier, leaving a stub behind.
3. **Version management** — expire noncurrent versions and clean up delete markers and
   incomplete multipart uploads.

MinIO implements the S3 lifecycle API, so rules are portable between MinIO and AWS S3 with
minor differences in tier configuration.

## Writing rules

### Via `mc ilm`

```bash
# Expire everything under logs/ after 30 days.
mc ilm rule add myminio/artifacts \
  --prefix "logs/" \
  --expire-days 30

# Keep only 3 noncurrent versions, and no version older than 14 days.
mc ilm rule add myminio/artifacts \
  --noncurrent-expire-days 14 \
  --noncurrent-expire-newer 3

# Clean up incomplete multipart uploads. Do this on EVERY bucket.
mc ilm rule add myminio/artifacts \
  --expire-delete-marker \
  --noncurrent-expire-days 14

mc ilm rule ls myminio/artifacts
```

### Via a JSON document

Preferred for anything version-controlled, because the whole policy is one reviewable
artifact.

```json
{
  "Rules": [
    {
      "ID": "expire-build-logs",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Expiration": {
        "Days": 30
      }
    },
    {
      "ID": "tier-cold-artifacts",
      "Status": "Enabled",
      "Filter": {
        "And": {
          "Prefix": "releases/",
          "Tags": [
            { "Key": "retention", "Value": "long" }
          ]
        }
      },
      "Transition": {
        "Days": 90,
        "StorageClass": "COLD-S3"
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30,
        "NewerNoncurrentVersions": 2
      }
    },
    {
      "ID": "housekeeping",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      },
      "Expiration": {
        "ExpiredObjectDeleteMarker": true
      }
    }
  ]
}
```

```bash
mc ilm import myminio/artifacts < lifecycle.json
mc ilm export myminio/artifacts    # confirm what is actually in effect
```

## The rules everyone forgets

### Incomplete multipart uploads

A multipart upload that is started and never completed or aborted leaves its parts on
disk, consuming space, invisible to `mc ls`, and counted against capacity. Backup tools
that get killed mid-upload produce these constantly.

```json
{
  "ID": "abort-stale-multipart",
  "Status": "Enabled",
  "Filter": {},
  "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
}
```

Apply this to every bucket, unconditionally. It is the single highest-value lifecycle rule
and the most commonly missing one. Find existing orphans with:

```bash
mc ls --incomplete --recursive myminio/artifacts
```

### Expired object delete markers

With versioning on, deleting an object creates a delete marker rather than removing
anything. When all real versions behind it have expired, the marker remains — a zero-byte
object that still costs a metadata entry and shows up in listings. `ExpiredObjectDeleteMarker: true`
removes markers with nothing behind them.

Note it is mutually exclusive with `Expiration.Days` in the same rule; use separate rules.

### Noncurrent version counts, not just days

`NoncurrentDays` alone is insufficient for a bucket with churn — an object rewritten every
minute accumulates thousands of versions inside the retention window. Combine both:

```json
"NoncurrentVersionExpiration": {
  "NoncurrentDays": 30,
  "NewerNoncurrentVersions": 5
}
```

Semantics: keep the 5 most recent noncurrent versions regardless of age; expire anything
older than 30 days *beyond* those 5.

## Tiering to cold storage

MinIO can transition object data to a remote tier while keeping the metadata local. Reads
of a transitioned object are transparent but slow — MinIO fetches from the remote on
demand.

### Configuring a tier

```bash
# S3 Glacier Instant Retrieval as a cold tier.
mc ilm tier add s3 myminio COLD-S3 \
  --access-key "$AWS_ACCESS_KEY_ID" \
  --secret-key "$AWS_SECRET_ACCESS_KEY" \
  --bucket cold-archive \
  --prefix minio-tier/ \
  --storage-class GLACIER_IR \
  --region eu-west-1

# Another MinIO cluster on cheaper spinning disks.
mc ilm tier add minio myminio COLD-LOCAL \
  --endpoint https://minio-archive.internal:9000 \
  --access-key "$ARCHIVE_KEY" \
  --secret-key "$ARCHIVE_SECRET" \
  --bucket archive

mc ilm tier ls myminio
mc ilm tier info myminio COLD-S3
```

Then reference the tier name as `StorageClass` in a `Transition` action.

### What tiering actually buys, honestly

It reduces the *storage* cost of cold data. It does not reduce complexity, and it
introduces several things that will surprise you:

- **Restore latency.** A transitioned object read is a remote fetch. For Glacier
  Flexible/Deep Archive, that is hours, and the S3 API returns an error until the restore
  completes. Only Instant Retrieval behaves like normal S3.
- **Egress and request costs.** Reading transitioned data back from a cloud tier costs
  money per GB and per request. A single misbehaving client that scans the whole bucket
  can generate a startling bill.
- **A new dependency.** Your object store now depends on another object store's
  availability and credentials. Rotating those credentials without breaking reads of
  already-transitioned data requires care — the tier config holds them, and objects
  already transitioned reference that tier by name.
- **It does not reduce object count.** Metadata stays local. If your problem is 500
  million tiny objects, tiering does not help; the metadata is the cost.

Tier when there is a genuine, measured cold tail — large objects, rarely read, kept for
compliance. Do not tier as a general capacity strategy; expiry is almost always the
cheaper answer, because the cheapest byte is the one you deleted.

### Do not tier your backups

A backup that lives behind a slow, differently-credentialed, cross-account remote fetch is
a backup you cannot restore under pressure. Keep the current retention window on local
capacity and tier only the long-tail compliance copies — and rehearse a restore *from the
tier* before relying on it.

## Object Lock and lifecycle interaction

Object Lock (WORM) prevents deletion until the retention period elapses. Lifecycle
expiration **cannot** remove a locked object — the rule runs, fails silently for that
object, and moves on. Consequences:

- Set the lifecycle expiry **longer** than the maximum lock retention, or objects
  accumulate forever.
- Object Lock must be enabled at bucket creation. It cannot be added later.
- With `COMPLIANCE` mode, not even the root account can delete before expiry. That is the
  point, and it is also how a mistyped retention of `36500` days becomes a permanent line
  item. Use `GOVERNANCE` mode unless there is a genuine regulatory requirement — it allows
  a privileged bypass with `s3:BypassGovernanceRetention`.

See `backup-target-patterns.md` for the retention design that goes with this.

## Verifying rules actually run

Lifecycle in MinIO is executed by the background scanner. It is not instantaneous, and on
a large bucket a full pass takes hours.

```bash
# What rules are in effect.
mc ilm rule ls myminio/artifacts

# Scanner activity and speed.
mc admin scanner status myminio 2>/dev/null || mc admin info myminio

# Did the data actually shrink?
mc du --versions myminio/artifacts
```

A useful habit: after adding an expiry rule, record `mc du` output, wait a scanner cycle,
and compare. Rules that silently do nothing — wrong prefix, wrong tag filter, a trailing
slash mismatch — are common and invisible until you check the number.

Common reasons a rule does nothing:

- The prefix has a leading slash. It should not: `logs/`, not `/logs/`.
- Tag filters use `And` when combined with a prefix; a bare `Tags` alongside `Prefix` at
  the same level is invalid.
- The rule is `Disabled`.
- Objects are under Object Lock retention.
- The scanner is starved — MinIO deprioritises it under load, so a permanently busy
  cluster expires slowly.

## Sensible defaults per bucket type

| Bucket purpose | Versioning | Expiry | Noncurrent | Multipart abort | Tier |
|---|---|---|---|---|---|
| Backups (Velero, pgBackRest) | On + Object Lock | Match retention policy, longer than lock | 30d / 3 versions | 1d | No |
| CI artifacts | Off | 30d | — | 1d | No |
| Application uploads | On | Never (user data) | 90d / 5 versions | 7d | Maybe, for the cold tail |
| Logs | Off | 14–30d | — | 1d | No |
| Compliance archive | On + Object Lock COMPLIANCE | 7y | — | 7d | Yes |

## Checklist

- [ ] Every bucket has an `AbortIncompleteMultipartUpload` rule
- [ ] Every versioned bucket has both `NoncurrentDays` and `NewerNoncurrentVersions`
- [ ] Delete-marker cleanup rule present
- [ ] Expiry set longer than any Object Lock retention on the same bucket
- [ ] Rules exported to version control (`mc ilm export`) and reviewed like code
- [ ] Effect verified with `mc du --versions` after a scanner cycle, not assumed
- [ ] Backups are not tiered, or the restore-from-tier path is rehearsed
