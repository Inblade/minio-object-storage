# Access Policies and Least Privilege

MinIO implements the S3 policy language: the same JSON documents, the same
`Effect`/`Action`/`Resource`/`Condition` structure, mostly the same evaluation semantics.
If you can write an S3 policy you can write a MinIO one, and the reverse is true — this
knowledge transfers.

## Identity model

Four things, in decreasing order of privilege:

1. **Root** (`MINIO_ROOT_USER`) — full control, bypasses policy. Used to bootstrap and
   then not used. It should live only in the secret manager and never in an application's
   configuration.
2. **Users** — created by root/admin, attached to one or more policies.
3. **Groups** — a policy attachment point for many users. Useful for humans; rarely for
   services.
4. **Service accounts (access keys)** — child credentials of a user, optionally with a
   *further-restricted* inline policy. This is what applications should use.

The important property of service accounts: their effective permission is the
**intersection** of the parent user's policy and the service account's own inline policy.
A service account can never exceed its parent. That makes them safe to hand out and cheap
to rotate — revoking one does not disturb the parent identity or any sibling key.

## Built-in policies

`readonly`, `readwrite`, `diagnostics`, `writeonly`, `consoleAdmin`. They exist for
convenience and are almost never the right answer, because none of them scope to a bucket.
`readwrite` means read-write on *everything*, including your backup bucket. Treat them as
examples, not as production policies.

## Policy examples

### Backup writer, append-only, single bucket

The shape you want for a backup agent: it can write new objects and list, but it cannot
delete. Combined with Object Lock, a compromised backup agent cannot destroy the backups.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOwnBucketOnly",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": ["arn:aws:s3:::pgbackrest"]
    },
    {
      "Sid": "WriteAndReadObjects",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListMultipartUploadParts",
        "s3:AbortMultipartUpload"
      ],
      "Resource": ["arn:aws:s3:::pgbackrest/*"]
    },
    {
      "Sid": "NoDeletes",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:PutBucketVersioning",
        "s3:PutObjectRetention",
        "s3:PutObjectLegalHold",
        "s3:BypassGovernanceRetention"
      ],
      "Resource": [
        "arn:aws:s3:::pgbackrest",
        "arn:aws:s3:::pgbackrest/*"
      ]
    }
  ]
}
```

The explicit `Deny` matters: deny always beats allow, so even if this identity later
acquires a broad `Allow` through another attached policy, deletion stays blocked. Expiry
of old backups is then handled by lifecycle rules running under MinIO's own authority, not
by the agent.

Note `s3:AbortMultipartUpload` is included — without it, a backup agent that fails partway
through cannot clean up its own parts, and those parts silently consume capacity.

### Prefix-scoped tenant, path isolation

Multiple teams sharing one bucket, each confined to its prefix. The `Condition` on
`ListBucket` is what stops a team from enumerating other teams' objects.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyOwnPrefix",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::artifacts"],
      "Condition": {
        "StringLike": {
          "s3:prefix": ["team-a/*", "team-a"]
        }
      }
    },
    {
      "Sid": "FullAccessWithinOwnPrefix",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": ["arn:aws:s3:::artifacts/team-a/*"]
    }
  ]
}
```

Without the `s3:prefix` condition, `ListBucket` on the bucket ARN lists the entire bucket —
object names alone often leak more than the objects would.

### Read-only consumer with a time bound

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": ["arn:aws:s3:::releases/*"],
      "Condition": {
        "DateLessThan": {
          "aws:CurrentTime": "2026-12-31T23:59:59Z"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::releases"]
    }
  ]
}
```

An expiry date on a policy is a cheap way to make temporary access actually temporary —
the most common credential problem is not a leaked key, it is a key issued for a migration
in 2024 that still works.

### Using policy variables for self-service

MinIO supports `${aws:username}` substitution, which lets one policy serve every user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::userdata"],
      "Condition": {
        "StringLike": {
          "s3:prefix": ["home/${aws:username}/*"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::userdata/home/${aws:username}/*"]
    }
  ]
}
```

One policy, attached to a group, and every member gets their own isolated home prefix.

### Enforcing encryption and TLS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedTransport",
      "Effect": "Deny",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::sensitive",
        "arn:aws:s3:::sensitive/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    },
    {
      "Sid": "RequireSSEOnUpload",
      "Effect": "Deny",
      "Action": ["s3:PutObject"],
      "Resource": ["arn:aws:s3:::sensitive/*"],
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

## Applying policies

```bash
# Create a named policy from a file.
mc admin policy create myminio pgbackrest-writer ./policies/pgbackrest-writer.json

# A user for the workload, then attach.
mc admin user add myminio pgbackrest-svc
mc admin policy attach myminio pgbackrest-writer --user pgbackrest-svc

# A service account (access key) for the actual application. This is the credential the
# Pod gets — rotate it without touching the user or its policy.
mc admin user svcacct add myminio pgbackrest-svc \
  --name "pgbackrest-prod-2026-08" \
  --description "pgBackRest repo1, prod cluster"

# Further restrict a service account below its parent's policy.
mc admin user svcacct add myminio pgbackrest-svc \
  --policy ./policies/single-stanza-only.json

# Audit.
mc admin policy ls myminio
mc admin policy info myminio pgbackrest-writer
mc admin user svcacct ls myminio pgbackrest-svc
```

## Bucket policies vs. IAM policies

Two places a policy can live, and they answer different questions:

- **IAM policy** (attached to a user/group) — "what may this identity do?"
- **Bucket policy** (attached to a bucket) — "who may do what to this bucket?", including
  anonymous access.

```bash
mc anonymous set download myminio/public-assets     # anonymous read
mc anonymous set-json ./bucket-policy.json myminio/public-assets
mc anonymous get-json myminio/public-assets
```

Anonymous bucket policies are the mechanism behind every "public S3 bucket" story. Audit
them regularly:

```bash
for b in $(mc ls myminio --json | jq -r '.key' | tr -d '/'); do
  printf '%s: ' "$b"
  mc anonymous get "myminio/$b"
done
```

Prefer IAM policies. Reach for bucket policies only when you genuinely need anonymous or
cross-identity access, and then scope them to a prefix.

## Evaluation order

1. An explicit `Deny` anywhere wins, always.
2. Otherwise, an explicit `Allow` grants.
3. Otherwise, denied by default.

This makes `Deny` statements a reliable guard rail: attach a broad protective deny at the
group level and no future policy addition can accidentally undo it.

## Credential delivery in Kubernetes

Do not put access keys in a ConfigMap, a Helm `values.yaml`, or the Tenant CR. Options in
rough order of preference:

1. **OIDC / STS** — MinIO can trust the cluster's ServiceAccount token issuer via
   `AssumeRoleWithWebIdentity`, so Pods get short-lived credentials with no static secret
   at all. Best option where the effort is justified.
2. **External Secrets Operator / Vault Agent** — the key lives in the secret manager and
   is synced to a Kubernetes Secret. Rotation is a change in one place.
3. **A sealed/SOPS-encrypted Secret in Git** — acceptable, rotate on a schedule.
4. A plain Secret created by hand — you will not remember which of them exist in a year.

## Rotation

Service accounts make rotation genuinely low-risk, because two can be valid at once:

```bash
# 1. Issue the new key.
mc admin user svcacct add myminio pgbackrest-svc --name "pgbackrest-prod-2026-09"

# 2. Roll it out to the workload; both keys work.
# 3. Confirm no traffic on the old key (MinIO audit log / access key in request logs).
# 4. Remove the old one.
mc admin user svcacct rm myminio OLDACCESSKEY
```

Step 3 is the one people skip, and it is the one that prevents a 3am outage. Keep the
audit log going somewhere queryable so "is this key still in use?" is a one-minute
question.

## Audit checklist

- [ ] Root credentials appear in exactly one place (the secret manager) and are unused by
      applications
- [ ] No identity holds a built-in `readwrite` policy in production
- [ ] Every application uses a service account, not a user's primary credentials
- [ ] Backup writers cannot delete (explicit `Deny` + Object Lock)
- [ ] No anonymous bucket policies except deliberately public buckets
- [ ] `ListBucket` grants are prefix-conditioned in shared buckets
- [ ] Service accounts named with owner and issue date so stale ones are identifiable
- [ ] Audit log shipped somewhere queryable
