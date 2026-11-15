---
name: object-storage
description: "S3-compatible object storage in 2026: MinIO is dead (repo archived, images deleted from Docker Hub). Use when writing Docker Compose dev stacks, S3 client code, presigned-URL features, or choosing object storage. Covers RustFS and Silo (MinIO fork) configs, env vars, ports, healthcheck paths, gotchas, and storage-agnostic client setup."
---

# MinIO is gone. Use RustFS or Silo.

Timeline: admin console stripped from community edition (May 2025) → no prebuilt binaries/Docker images (Oct 2025) → maintenance mode (Dec 2025) → repo archived (Apr 2026) → `minio/minio` and `minio/mc` **deleted from Docker Hub** (Sept 11, 2026). `image: minio/minio` fails to pull; the frozen quay.io mirror misses an open CVSS-8.1 CVE.

## Stale → current

| Stale (pre-2026 models) | Current |
| --- | --- |
| `image: minio/minio` | `rustfs/rustfs:1.0.5` or `pgsty/silo:RELEASE.2026-11-12T00-00-00Z` |
| `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` | RustFS: `RUSTFS_ACCESS_KEY` / `RUSTFS_SECRET_KEY` (Silo keeps `MINIO_*` vars) |
| Health at `/minio/health/live` | RustFS: `/health`, `/health/live`, `/health/ready` (`/minio/health/*` aliases exist); Silo: unchanged |
| Console at bare `:9001` | RustFS console: `http://localhost:9001/rustfs/console/`; Silo: unchanged |
| `mc` client everywhere | Silo: `mc` compatible. RustFS: ships its own `rc` CLI |

## Pick one

| | RustFS 1.0.5 (Nov 2026) | Silo (Pigsty MinIO fork) |
| --- | --- | --- |
| License | Apache-2.0 | AGPL-3.0 |
| Codebase | Rust rewrite | MinIO Go code, maintained |
| MinIO drop-in fidelity | S3 subset + some behavioral deltas | Exact: env vars, `/minio/*` routes, `mc`, on-disk format, console |
| Versioning / Object Lock | Yes | Yes (as MinIO) |
| Choose when | New projects, license-sensitive redistribution | Existing MinIO deploys; zero-delta dev parity |

Know these exist: **Garage** (AGPL, lean single binary, ports 3900+, no versioning/Object Lock/console — dumb byte store) and **SeaweedFS** (Apache-2.0, versioning + Object Lock, but master/filer/S3 topology — heavier ops).

## RustFS dev compose

```yaml
services:
  rustfs:
    image: rustfs/rustfs:1.0.5
    environment:
      RUSTFS_ACCESS_KEY: dev-key           # defaults rustfsadmin/rustfsadmin
      RUSTFS_SECRET_KEY: dev-secret
      RUSTFS_ADDRESS: ":9000"              # S3 API (default)
      RUSTFS_CONSOLE_ENABLE: "true"        # default true
      RUSTFS_CONSOLE_ADDRESS: ":9001"      # default
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - rustfs_data:/data                  # default volume path (no RUSTFS_VOLUMES needed for dev)
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:9000/health/ready"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 30s
volumes:
  rustfs_data:
```

Gotchas:
- `/health/ready` returns **503 until storage/IAM is fully up** — always set `start_period`. `/health` can report ready while `/health/ready` still 503s in the same second (rustfs#6285); readiness is cached for 1s.
- Console URL is `http://localhost:9001/rustfs/console/` — not bare `:9001`.
- Presigned URLs bind to the exact host/scheme/port they were minted for.
- Mixed 1.0.4/1.0.5 clusters can return HTTP 503 on DELETE — upgrade all nodes together.
- `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` are accepted as deprecated aliases (mapped to `RUSTFS_*`).

## Silo dev compose (exact MinIO behavior)

```yaml
services:
  silo:
    image: pgsty/silo:RELEASE.2026-11-12T00-00-00Z
    command: server /data --console-address ":9001"   # legacy `minio server /data` argv also works
    environment:
      MINIO_ROOT_USER: dev-key
      MINIO_ROOT_PASSWORD: dev-secret
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - silo_data:/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:9000/minio/health/ready"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 30s
volumes:
  silo_data:
```
The classic image has **no baked-in HEALTHCHECK**; the `-distroless` variant has one (`silo healthcheck ready`). `mc` works as-is; console on `:9001` unchanged from MinIO.

## Keep clients storage-agnostic

Point a generic S3 client at whatever endpoint dev/prod provides; never import server-specific SDKs.

Rust — `object_store` (0.14) against a local endpoint:

```rust
use object_store::aws::AmazonS3Builder;

let store = AmazonS3Builder::new()
    .with_endpoint("http://127.0.0.1:9000")   // RustFS / Silo / MinIO-style endpoint
    .with_bucket_name("media")
    .with_access_key_id(&key)
    .with_secret_access_key(&secret)
    .with_region("us-east-1")
    .with_allow_http(true)      // REQUIRED: default is HTTPS-only, plain http:// is rejected
    .with_virtual_hosted_style_request(false)   // path-style (default) is what these servers want
    .build()?;
```
`AmazonS3Builder::from_env()` works too (`AWS_ENDPOINT`, `AWS_ALLOW_HTTP=true`, `AWS_ACCESS_KEY_ID`, ...). The same client talks to AWS S3 in prod.

Go — `aws-sdk-go-v2` S3 client with a custom `BaseEndpoint` (or `EndpointResolverV2`) works against all of them; any S3 tool (rclone, aws cli `--endpoint-url`) too.

## Sources

- https://github.com/rustfs/rustfs/releases/tag/1.0.5 (2026-11-10)
- https://docs.rustfs.com/en/reference/environment-variables
- https://docs.rustfs.com/en/operations/status-check
- https://docs.rustfs.com/en/reference/s3-compatibility
- https://silo.pgsty.com/compatibility/versions · https://silo.pgsty.com/compatibility/server (2026-08-31)
- https://github.com/pgsty/silo · https://hub.docker.com/r/pgsty/silo
- MinIO timeline: https://stablebuild.com/blog/minio-images-disappeared-from-docker-hub · https://vonng.com/en/db/silo-is-coming · https://lwn.net/Articles/1077739
