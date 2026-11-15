---
name: rust-web-api
description: "Current Rust backend web stack (Nov 2026): Axum 0.8 breaking changes, SQLx 0.9 workflow, utoipa 6 OpenAPI 3.1, tower-http 0.7. Use when writing Rust HTTP handlers, extractors, compile-time-checked SQL queries, or OpenAPI-generating APIs. Corrects pre-0.8 Axum route syntax, unnecessary #[async_trait], stale SQLx guidance, and wrong crate versions."
---

# Rust web stack — versions and breaking changes (Nov 2026)

Current crates: axum **0.8.9** (0.9 is NOT released — don't write `axum = "0.9"`), sqlx **0.9.0**, utoipa **6.0** (OpenAPI 3.1 only), utoipa-swagger-ui **10.x**, tower-http **0.7.1**, object_store **0.14**, lettre **0.11**, Rust stable **1.100** (MSRV floor for these crates: 1.80–1.94).

## Axum 0.8 breaking changes (0.7 → 0.8)

| Stale (pre-2026 models) | Current (0.8) |
| --- | --- |
| `.route("/users/:id", ...)` | `.route("/users/{id}", ...)` — old syntax **panics at router construction** |
| `/*rest` catch-all | `{*rest}`; literal braces escape as `{{` / `}}` |
| `#[async_trait]` on custom extractors/handlers | Removed — native RPITIT (Rust 1.75+); just delete the attribute |
| `Option<Query<T>>` to ignore errors | Broken — `OptionalFromRequestParts` impl for `Query` was removed; parse and handle the `Query` error explicitly |
| `Rc`/`RefCell` in shared state | Handlers/state added to `Router` must now be `Sync` |

```rust
let app = Router::new()
    .route("/items/{item_id}", get(get_item))
    .route("/items/{item_id}/usage/{usage_id}", get(get_usage))
    .route("/assets/{*path}", get(serve_asset));
```
Also changed: WebSocket `Message` payloads are `Bytes`/`Utf8Bytes` (not `Vec<u8>`/`String`); `Host` extractor moved to `axum-extra`; `serve` is generic over the listener; MSRV 1.80 (0.8.9).

## SQLx 0.9 (May 2026 — post many cutoffs)

- Offline workflow unchanged: `cargo sqlx prepare` → commit the `.sqlx` dir, `SQLX_OFFLINE=true` in Docker builds, `#[sqlx::test]` for integration tests. `SQLX_OFFLINE_DIR` now takes precedence in macros.
- Repo moved to **github.com/transact-rs/sqlx** (no longer LaunchBadge).
- New `sqlx.toml` config file (rename `DATABASE_URL` per crate, global macro type overrides, relocate `_sqlx_migrations`).
- `query*()` functions now take `impl SqlSafeStr` — only `&'static str` implements it. Dynamic/owned-SQL strings must be wrapped: `sqlx::query(sqlx::AssertSqlSafe(...))`. (This is the mechanism that allows owned `String` queries.)
- `cargo install --locked sqlx-cli` no longer works (Cargo.lock untracked) — plain `cargo install sqlx-cli`.
- SQLite: `journal_mode` is **not** set by default (set `.journal_mode(SqliteJournalMode::Wal)` explicitly); `foreign_keys` on by default; `locking_mode` default NORMAL.

## utoipa 6

- Generates **OpenAPI 3.1 only** — 3.0 output was dropped in 5.0. Old snippets producing 3.0 fields are stale.
- `utoipa-swagger-ui` is at 10.x; the `vendored` feature moved to the external crate `utoipa-swagger-ui-vendored`.

## tower-http 0.7

Standard middleware set (feature-gated): `TraceLayer`, `CorsLayer`, `CompressionLayer`/`DecompressionLayer` (gzip, br, zstd), `TimeoutLayer`, `RequestIdLayer`/`SetRequestIdLayer`/`PropagateRequestIdLayer`, `CatchPanicLayer`, `NormalizePathLayer`, `ServeDir`, `LimitRequestBody`, `ValidateRequestLayer`. New in 0.7: `CsrfLayer`.

## object_store 0.14 — local S3 endpoint

S3/Azure/GCP impls moved to sub-crates (`object_store_aws`, ...), re-exported via the facade. For pointing at a local RustFS/Silo endpoint (the `with_allow_http(true)` requirement), see the `object-storage` skill.

## Sources

- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0 · https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md (0.8.9, 2026-04-14)
- https://github.com/transact-rs/sqlx/blob/main/CHANGELOG.md (0.9.0, 2026-05-21) · https://docs.rs/sqlx/latest/sqlx/sqlite/struct.SqliteConnectOptions.html
- https://docs.rs/utoipa/latest/utoipa/ · https://github.com/juhaku/utoipa/discussions/1124 (3.1-only)
- https://docs.rs/object_store/latest/object_store/aws/struct.AmazonS3Builder.html (0.14.2)
- https://crates.io/crates/tower-http (0.7.1, 2026-08-31)
