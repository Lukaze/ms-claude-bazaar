---
name: rust-coding
description: "Use when editing or creating `*.rs` files, porting Python to Rust, or deciding on error handling, async patterns, traits, tracing, or concurrency in a Rust project."
---

# Rust Coding Skill

Opinionated Rust rules. Priority order:

1. **Readable code** — every line should earn its place
2. **Correct code** — especially in concurrent/async contexts
3. **Performant code** — think about allocations, data structures, hot paths

Reasonable defaults for a modern workspace: **edition 2024**, a recent stable `rust-version`, and a workspace-level `unsafe_code = "deny"` lint. Do not silence `deny`ed lints — raise the design question first.

## Foundation: The Rust Book

When unsure about the idiomatic Rust approach, **fetch and read the relevant chapter** from [The Rust Programming Language](https://doc.rust-lang.org/stable/book/) before proceeding. Key chapters: ownership (ch 4), enums/matching (ch 6), error handling (ch 9), generics/traits/lifetimes (ch 10), closures/iterators (ch 13), smart pointers (ch 15), concurrency (ch 16).

**When porting Python to Rust, do not transliterate Python patterns.** Common traps:
- Python classes with inheritance → Rust enums + traits (not struct inheritance hacks)
- Python `dict` everywhere → typed structs with `Option` fields (except at open-schema boundaries — see below)
- Python `try/except` catch-all → `Result` with typed error enums propagated via `?`
- Python mutable shared state → ownership model; if sharing is needed, `Arc<Mutex<T>>` or channels
- Python `None` checks → `Option` with `map`, `and_then`, `unwrap_or_else`

## Error Handling

| Rule | Do | Don't |
|------|----|-------|
| Module / crate boundaries | `thiserror` enum (per-domain error type) | `Box<dyn Error>` |
| Binary / top-level only | `anyhow::Result` in `main`, integration tests | `anyhow` in library crates |
| Production code | `?`, `let-else`, `if let`, `expect("reason")` | `unwrap()` (tests only) |
| Log-and-propagate | `inspect_err` + `warn!`/`error!`, then `?` | `match` that logs and re-wraps |

## Async & Concurrency

| Rule | Do | Don't |
|------|----|-------|
| Request-scoped I/O | `async fn` + `.await` (futures) | `tokio::spawn` per request |
| Background work | `tokio::spawn` + track `JoinHandle`; attach a span | Fire-and-forget spawn with no span / no handle |
| Blocking I/O (fs, subprocess) | `tokio::task::spawn_blocking` | Blocking the async runtime |
| Long-lived workers | `std::thread::spawn` | `spawn_blocking` (pool is bounded) |
| Locks held across `.await` | `tokio::sync::Mutex` / `RwLock` | `std::sync::Mutex` / `parking_lot::Mutex` |
| Hot-path sync locks | `parking_lot::Mutex` | `std::sync::Mutex` |
| Lazy statics | `std::sync::LazyLock` (edition 2024) | `once_cell::Lazy` |
| Parallel awaits | `tokio::try_join!` / `futures::future::try_join_all` | `for`-loop of awaits when work is independent |

### Single-flight, not re-implemented

When a component needs to coalesce concurrent requests (e.g., token resolution, cache fills), use a shared in-flight map: `HashMap<Key, Shared<BoxFuture<Result<T>>>>` guarded by a mutex. Each caller either creates the future or awaits the existing one — don't let N concurrent callers race to do the same work N times.

## Traits & Conversions

| Rule | Do | Don't |
|------|----|-------|
| Trait usage | Plain functions / inherent methods on the type | Speculative traits ("for testing later") — they break code navigation |
| Trivial field mapping | Construct struct inline at call site | Free-standing `map_x_to_y()` functions |
| Reusable conversion | Named method: `into_bar(self)`, `to_info(&self)`, `MyType::from_record(r)` | `From`/`Into` (can't express extra params or context) |
| Closures | Keep <10 lines; extract to a named fn if larger | Long anonymous closures (invisible in stack traces) |
| Visitor pattern | Extract traversal into an `iter()` method | Trait-based visitors |

Traits are appropriate at deliberate extension points (e.g., a connector trait with multiple real implementations, or pluggable strategies under test). Default to inherent methods otherwise.

## Tracing

Structured tracing with `tracing` + span propagation. Hard rules:

- **Almost always use `error_span!`**, not `info_span!`. Span level is the *minimum* filter at which the span appears; an `info_span` disappears when the filter is `warn`/`error`, taking child events with it — including errors.
- **Spawned tasks lose parent context.** Always attach a span via `.instrument()` so events inside correlate back to the originating request.
- **Never hold `span.enter()` guards across `.await`.** Use `.instrument(span)` instead.
- **Do not annotate (async) generators / streams with span guards** — wrap the awaited body in `.instrument(span)` instead.
- Keep `#[tracing::instrument]` use disciplined: prefer manual `error_span!` so field names and parents are explicit. If you do use the macro, set `level = "error"` and `skip_all` by default.

```rust
use tracing::Instrument;

async fn send_message(&self, session_id: &str, messages: Vec<Message>) -> Result<()> {
    let span = tracing::error_span!(
        "send_message",
        session_id = %session_id,
        count = messages.len(),
    );
    async { /* body */ }.instrument(span).await
}

// Spawned tasks need explicit span attachment to keep the trace chain intact.
let span = tracing::error_span!("worker", session_id = %id);
tokio::spawn(async move { work().await }.instrument(span));
```

Log with structured fields, not interpolation: `info!(session_id = %id, "Session created")`. Static messages stay greppable; dynamic data goes in named fields.

## Data Modeling

### Pure-data crates stay pure

If a crate's purpose is to hold shared types (e.g., a `domain` or `model` crate), keep it free of I/O, network clients, and async runtime dependencies. Types are cheap; side effects belong in service crates.

### Open-schema bags are intentional

Some boundaries deal with open-schema payloads from LLMs / MCP / OpenAPI / Swagger connectors. `HashMap<String, serde_json::Value>` is the **intentional** representation in those builders. Do not "tighten" them into structs — the schema is provider-driven and drops unknown fields silently if you do.

Inverse rule: persisted domain models keep typed fields. `serde_json::Value` is only acceptable method-locally for serialization output, never as a stored field.

### Shared clients

Share long-lived clients (`reqwest::Client`, tonic channels). Construct once at startup and inject; never build a new `reqwest::Client` per request — it bypasses connection pooling and DNS caching.

### Protos

If you use protobuf / gRPC, keep one canonical location for `.proto` files (typically a repo-root `proto/` directory). Do not vendor duplicated `.proto` files into service crates. Generated code goes through a `build.rs` or a regeneration script — pick one approach and stick with it.

## Code Organization

- **Prefer `pub` over `pub(crate)` in private modules.** Most submodules are declared `mod` (private), so `pub` items are already crate-private. Use `pub(crate)` only inside `pub mod` modules where items must not leak to the public API.
- **`#[expect(dead_code)]`** instead of `#[allow(dead_code)]` on fields you intend to be unused — `expect` fails the build if the lint stops firing.
- **`..Default::default()`** — avoid in production paths (be explicit about every field); fine in tests to reduce boilerplate.
- **Import grouping** — three blocks separated by blank lines: (1) `std`/`core`/`alloc`, (2) external crates, (3) `crate::`/`super::`/`self::`. Enforced by `cargo fmt`.
- **No `unsafe`.** If the workspace denies it, keep it denied; if you genuinely need it, raise the design question first rather than adding `#[allow(unsafe_code)]`.

## Testing

- **Avoid mock testing.** Depend on real implementations, spin up lightweight versions (in-memory stores, real `tokio` runtime, ephemeral temp dirs), or restructure code so logic takes dependency *output* as input.
- **`assert_eq!(actual, expected)`** — actual first for readable diffs.
- **`#[cfg(test)] mod tests` at end of file.** Never place production code after the test module.
- **Concurrent-safe tests.** Unique temp dirs (`tempfile::tempdir()`), unique key prefixes, unique ports. Use `serial_test` only when truly necessary — it serializes the whole suite and hides flakiness.
- **Async tests.** `#[tokio::test(flavor = "multi_thread")]` when you need to assert behavior under real concurrency; the default current-thread flavor masks deadlocks.

## Comments

- Explain **why**, never **what**. No comments that restate code.
- No decorative banner / divider comments (`// ── Section ────────`).
- Doc comments (`///`) on public items in library crates; module-level `//!` doc on each `mod` file describing its purpose.

## Build, Lint, Format

| Action | Command |
|---|---|
| Fast feedback | `cargo check --workspace --all-targets` |
| Lint (must be clean) | `cargo clippy --workspace --all-targets -- -D warnings` |
| Format check | `cargo fmt --all -- --check` |
| Tests | `cargo test --workspace --all-targets` |
| Single crate | `cargo test -p <crate_name>` |

Iterate with `cargo check`, not `cargo build`. Minimize Tokio features (do **not** use `features = ["full"]`); enumerate only what's used. Audit feature flags with `cargo tree -e features` before adding a dependency.
