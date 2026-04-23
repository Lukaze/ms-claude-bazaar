# Rust Coding Skill — Examples

Patterns that aren't obvious from general Rust knowledge.

## Span attachment on spawned tasks

### Anti-pattern — spawn loses the parent span
```rust
// Events from this task have no parent span; traces/logs can't correlate
// them back to the originating request.
tokio::spawn(async move {
    do_background_work(state).await;
});
```

### Preferred — attach an `error_span!` so the trace chain survives
```rust
use tracing::Instrument;

let span = tracing::error_span!("background_work", session_id = %session_id);
tokio::spawn(async move {
    if let Err(err) = do_background_work(state).await {
        tracing::warn!(error = %err, "background work failed");
    }
}.instrument(span));
```

## Lock discipline across `.await`

### Anti-pattern — `parking_lot` mutex held across an await
```rust
let guard = state.cache.lock(); // parking_lot::Mutex
let value = fetch_remote(&guard.key).await?; // 💀 holds lock across .await
guard.entries.insert(value.id, value);
```

### Fix — drop the lock, do the await, reacquire
```rust
let key = { state.cache.lock().key.clone() };
let value = fetch_remote(&key).await?;
state.cache.lock().entries.insert(value.id, value);
```

If the section truly needs to be atomic across the await, switch to `tokio::sync::Mutex` (or restructure so the await happens outside the critical section).

## Single-flight, not racing concurrent callers

### Anti-pattern — every caller races to mint the same token
```rust
pub async fn token_for(&self, identity: &Identity) -> Result<Token> {
    if let Some(t) = self.cache.get(identity) { return Ok(t); }
    let t = mint_token(identity).await?; // N concurrent callers => N exchanges
    self.cache.insert(identity.clone(), t.clone());
    Ok(t)
}
```

### Preferred — coalesce via an in-flight map of shared futures
Stash a `Shared<BoxFuture<Result<Token>>>` keyed by identity so concurrent callers await the same exchange. The first caller drives the work; the rest clone the `Shared` future and await the same result.

## Open-schema bag vs typed model

### Anti-pattern — tightening an LLM tool-call payload into a struct
```rust
// MCP/LLM tool arguments are provider-driven; the schema is open.
struct ToolArgs { query: String, top_k: u32 } // brittle, drops unknown fields
```

### Preferred — keep the bag at the boundary
```rust
use std::collections::HashMap;
use serde_json::Value;

pub struct ToolApprovalRequest {
    pub tool_name: String,
    pub parameters: HashMap<String, Value>,
}
```

### Inverse — persisted domain models stay typed
```rust
// Storage model: every persisted field has a real type.
pub struct WebhookSubscription {
    pub id: SubscriptionId,
    pub resource: String,
    pub expires_at: DateTime<Utc>,
    // serde_json::Value would be a smell here.
}
```

## `error_span!` over `info_span!`

### Anti-pattern — span disappears under `RUST_LOG=warn`
```rust
let span = tracing::info_span!("send_message", session_id = %id);
async { /* errors logged inside have no parent span */ }.instrument(span).await
```

### Preferred — span is always present, errors keep their parent
```rust
let span = tracing::error_span!("send_message", session_id = %id);
async { /* … */ }.instrument(span).await
```
