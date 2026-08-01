# MinIO Object Storage

Reference notes on running S3-compatible object storage on Kubernetes with MinIO: erasure
coding and capacity math, lifecycle and tiering, access policy design, using it as a
backup target, and the alerts worth waking someone for.

Written from operating MinIO as an in-cluster backup target and internal S3 endpoint —
the kind of deployment where the object store is infrastructure other systems depend on,
so its failure modes are everyone's failure modes. Employer-neutral and generalised; the
manifests are illustrative reference material rather than a drop-in deployment.

## Contents

```
.
├── docs
│   ├── erasure-coding.md          # parity math, capacity planning, failure tolerance
│   ├── lifecycle-and-tiering.md   # ILM rules, expiry, transition to cold storage
│   ├── access-policies.md         # bucket policies, service accounts, least privilege
│   └── backup-target-patterns.md  # Velero, pgBackRest, retention and immutability
├── k8s
│   └── minio-tenant.yaml          # MinIO Operator Tenant CR
├── monitoring
│   └── minio-alerts.yaml          # PrometheusRule
├── LICENSE
└── README.md
```

## How to use this

The order that matches how these decisions are actually made:

1. **`docs/erasure-coding.md`** — decide the drive layout and parity level first. It is
   fixed at tenant creation and determines both usable capacity and how many failures you
   survive. Getting this wrong is expensive to undo.
2. **`k8s/minio-tenant.yaml`** — the Tenant CR that encodes that decision, with the pool
   topology, resources, and anti-affinity that keep the erasure set honest.
3. **`docs/access-policies.md`** — no application should hold the root credentials. Set
   up per-workload service accounts with scoped policies before anything writes data.
4. **`docs/lifecycle-and-tiering.md`** — object storage grows until told not to. Write
   the expiry rules on day one.
5. **`docs/backup-target-patterns.md`** — if this is a backup target, object lock and
   retention design matters more than throughput.
6. **`monitoring/minio-alerts.yaml`** — deploy alongside the tenant, not after the first
   incident.

```bash
# Operator first (version-pin in real use).
kubectl apply -k "github.com/minio/operator?ref=v7.0.1"

# Then the tenant. Review storage classes and sizes before applying.
kubectl apply -f k8s/minio-tenant.yaml

# Alerts (requires the Prometheus Operator CRDs).
kubectl apply -f monitoring/minio-alerts.yaml
```

## Assumptions

- Kubernetes 1.28+, MinIO Operator v7.x (`minio.min.io/v2` Tenant API).
- `mc` (MinIO Client) available locally for policy and lifecycle administration.
- Prometheus Operator installed for the `PrometheusRule`.
- Local NVMe or a comparable direct-attached class for the data volumes. Running MinIO on
  network block storage works but pays for replication twice — see `erasure-coding.md`.

## License

MIT — see [LICENSE](LICENSE).
