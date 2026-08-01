# Erasure Coding and Capacity Planning

## The mechanism

MinIO splits every object into `k` data shards and `m` parity shards using Reed-Solomon
coding, and writes all `k + m` shards across distinct drives in an **erasure set**. Any
`k` of the `k + m` shards can reconstruct the object. So the tenant tolerates the loss of
`m` drives per erasure set with zero data loss.

This is not replication. Three-way replication stores 3x the bytes to survive two
failures. Erasure coding with EC:4 over 16 drives stores 1.33x the bytes to survive four.
That ratio is the entire reason to use it.

## Erasure sets

The set — not the whole cluster — is the unit of failure tolerance. MinIO chooses the set
size automatically from the drive count in a pool, preferring a size between 4 and 16 that
divides evenly.

| Drives in pool | Set size chosen | Sets |
|---|---|---|
| 4 | 4 | 1 |
| 8 | 8 | 1 |
| 12 | 12 | 1 |
| 16 | 16 | 1 |
| 32 | 16 | 2 |
| 48 | 16 | 3 |
| 64 | 16 | 4 |

Each object lands entirely within one set, chosen deterministically by hashing its name.
Consequences worth internalising:

- **Failures are counted per set, not per cluster.** A 64-drive tenant with EC:4 (four
  sets of 16) survives four failures *in the same set*. Four failures spread one per set
  is also survivable. Five in one set is data loss for the objects hashed there — while
  the other three quarters of the tenant remains fully healthy. Partial loss, not total.
- Larger sets give better storage efficiency at the same parity, and worse per-object
  write fan-out. 16 is a good default and MinIO's own preference.

## Parity levels

Set at tenant creation via the `MINIO_STORAGE_CLASS_STANDARD` environment variable
(`EC:2`, `EC:4`, `EC:8`, ...). The default is `EC:4` for sets of 8 or more.

For a 16-drive erasure set:

| Parity | Data:Parity | Usable capacity | Drive failures tolerated | Read quorum | Write quorum |
|---|---|---|---|---|---|
| `EC:2` | 14:2 | 87.5% | 2 | 14 | 15 |
| `EC:4` | 12:4 | 75.0% | 4 | 12 | 13 |
| `EC:6` | 10:6 | 62.5% | 6 | 10 | 11 |
| `EC:8` | 8:8 | 50.0% | 8 | 8 | 9 |

Two quorum numbers, and the difference matters:

- **Read quorum = `k`.** With `m` drives down, reads still succeed.
- **Write quorum = `k + 1`** (except at `EC:8`, where it equals `k` because otherwise the
  cluster could not tolerate its stated failure count for writes).

So a tenant can go read-only before it goes down. At `EC:4` on 16 drives: 4 drives down =
degraded but fully functional; 5 down = reads succeed, **writes fail**; 6 down = the set
is offline. An incident where "backups are failing but restores work" is exactly this.

### Choosing

- **`EC:2`** — only for large sets where you have fast replacement and the data is
  reconstructible from elsewhere. Two concurrent failures during a rebuild is not a
  far-fetched scenario.
- **`EC:4`** — the default and the right answer for most deployments. 75% efficiency,
  survives four drives.
- **`EC:8`** — for a backup target of record, or single-rack deployments where a rack
  event takes many drives at once. You pay half your raw capacity for it.

Parity can be raised per bucket-class later via `MINIO_STORAGE_CLASS_RRS` for
reduced-redundancy objects, but the STANDARD class parity for existing data is fixed —
changing it means a new tenant and a migration.

## Capacity math

Work in this order. Every step rounds *down* usable capacity, and skipping one is how
people end up 30% short.

```
raw            = drives × drive_size
after_parity   = raw × k / (k + m)
after_headroom = after_parity × 0.80        # never plan to run object storage full
usable         = after_headroom / versioning_and_dm_overhead
```

### Worked example

Target: 500 TiB usable for a backup target with 30-day retention.

```
Choose EC:8 (backup of record, want to survive a rack event)  → efficiency 0.50
Headroom factor                                                → 0.80
Versioning overhead (backups: delete markers + a little churn) → 1.15

raw_needed = 500 / (0.50 × 0.80) × 1.15
           = 500 / 0.40 × 1.15
           = 1437.5 TiB raw
```

With 16 TiB drives that is ~90 drives, so 96 drives (6 sets of 16) across 6 nodes at 16
drives each. Note how the "500 TiB" request became 1.4 PiB of purchased disk. Present that
arithmetic *before* the hardware is ordered, not after.

### The 80% rule

Do not plan past 80% utilisation:

- Healing a failed drive needs space to write the reconstructed shards.
- Multipart uploads reserve space before completing.
- Deletes with versioning enabled *increase* usage until the lifecycle rule expires the
  old versions.
- A full MinIO tenant fails writes hard, and every dependent backup job fails with it.

Alert at 75%, page at 85%. Expansion is possible but not instant (see below).

### Versioning overhead

Versioning is not free and is not proportional to your intuition. Each overwrite keeps the
old version at full size. A 1 GiB file overwritten nightly with 30-day version retention
occupies 30 GiB. Delete markers are small but numerous.

For a backup target with immutable, write-once objects, overhead is close to 1.0. For a
bucket backing a chatty application that rewrites objects, it can be 5x or worse. Measure
with `mc du --versions` rather than guessing.

## Drives, nodes, and what actually fails

### Drive layout

- **One drive per volume, `XFS`, no RAID.** MinIO wants raw drives. Putting erasure coding
  on top of RAID means paying for redundancy twice and giving MinIO a false picture of the
  failure domain — a RAID controller failure takes every "drive" at once.
- **Direct-attached NVMe or SAS.** MinIO's performance model assumes local disks.
- **Uniform drive sizes.** MinIO uses the smallest drive's capacity for all of them in a
  set. One 4 TiB drive among fifteen 16 TiB drives wastes 180 TiB.

### Running on network block storage

Common on managed Kubernetes where local NVMe is not available. It works, and it is a
compromise worth naming out loud: EBS/PD are already replicated internally, so MinIO's
parity is redundancy on top of redundancy — you are paying twice and getting worse latency
than local disks. It is defensible when the alternative is no object storage at all, or
when you deliberately want the operational simplicity of a volume that survives node
replacement. It is not defensible as a performance tier.

If you do it: use `EC:2` (the underlying storage already handles drive failure; parity is
now protecting against *node* failure), and spread across AZs with topology constraints.

### Node distribution

The erasure set should span as many nodes as possible so a node loss costs one drive per
set rather than all of them. With 4 nodes × 4 drives = 16 drives in one set, losing a node
loses 4 drives — exactly the `EC:4` tolerance, with nothing left over. That is a cluster
that survives a node failure and *nothing else*.

Pin it down with anti-affinity and topology spread in the Tenant CR, and size parity
against the **node** failure domain, not the drive one:

```
parity ≥ drives_per_node × (nodes_you_want_to_survive_losing)
```

## Healing

MinIO heals continuously in the background (`mc admin heal`, plus an automatic scanner).
Points that matter operationally:

- Healing reads from surviving shards and writes reconstructed ones — it consumes both IO
  and free space. A cluster at 95% may be unable to heal at all.
- Replacing a drive means presenting an empty, formatted XFS filesystem at the same mount
  path. MinIO detects it and heals into it. Do not copy old data back manually.
- Heal progress is visible:

  ```bash
  mc admin heal --recursive --dry-run myminio
  mc admin info myminio
  ```

- Healing a 16 TiB drive is measured in hours to days depending on network and load. Your
  parity level must cover a second failure *during that window*. This is the practical
  argument against `EC:2` on large drives.

## Expansion

MinIO expands by **adding a pool**, not by adding drives to an existing one. A new pool is
a new set of servers with their own erasure sets; the tenant then spans both.

```yaml
  pools:
    - name: pool-0
      servers: 4
      volumesPerServer: 4
      # ...
    - name: pool-1        # added later
      servers: 4
      volumesPerServer: 4
```

Behaviour to plan for:

- **Existing objects do not move.** New writes are distributed across pools weighted by
  free space, so the new pool takes most new data until they even out. Read performance
  for old data is unchanged.
- **Pools are independent failure domains** with their own parity. A pool can be
  decommissioned (`mc admin decommission start`), which drains its objects to the others —
  that is the supported way to retire old hardware.
- You cannot shrink a pool or change its parity. Plan pool geometry as a long-lived
  decision.

## Quick reference

```bash
# Cluster state: drives online/offline per set, capacity, healing.
mc admin info myminio

# Per-bucket usage including all versions.
mc du --versions myminio/backups

# What parity is actually in effect.
mc admin config get myminio storage_class

# Heal status.
mc admin heal --recursive myminio
```

## Checklist before creating a tenant

- [ ] Parity chosen against the **node** failure domain, not just drive count
- [ ] Drive count divides cleanly into sets of 16 (or a deliberate smaller set size)
- [ ] Uniform drive sizes
- [ ] Raw capacity computed with parity, 80% headroom, and versioning overhead
- [ ] Anti-affinity so a node loss costs one drive per set
- [ ] Rebuild window understood — parity covers a second failure during a heal
- [ ] Expansion plan is "add a pool", and the geometry of that pool is sketched
