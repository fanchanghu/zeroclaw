# [Feature]: Decouple memory strategy layer from storage backend via MemoryStrategy trait

**Labels**: `enhancement`

---

### Summary

Introduce a `MemoryStrategy` trait to decouple high-level memory lifecycle policies (context loading, consolidation, governance) from the low-level `Memory` storage backend, enabling pluggable retrieval and consolidation strategies without modifying backends.

### Problem statement

The current `Memory` trait (`crates/zeroclaw-api/src/memory_traits.rs`) is strictly a CRUD interface. All higher-order memory behaviors are scattered across the runtime:

- **Context loading** is duplicated between `DefaultMemoryLoader` (`crates/zeroclaw-runtime/src/agent/memory_loader.rs`) and `build_context()` (`crates/zeroclaw-runtime/src/agent/loop_.rs:375`). Both apply time decay, filter autosave keys, and format `[Memory context]` — a backend cannot customize retrieval strategy (e.g., knowledge-graph multi-hop, importance-boosted ranking).
- **Consolidation** is implemented in `crates/zeroclaw-memory/src/consolidation.rs` but triggered externally by the WebSocket gateway and channel orchestrator via fire-and-forget `tokio::spawn`. `Agent::turn()` (library path) never triggers consolidation, so the same conversation yields different memory growth depending on the entry point.
- **Governance/hygiene** is a standalone procedural function (`hygiene::run_if_due`) invoked during memory creation. Backends cannot define their own cleanup or background-integration ("dreaming") cadence.
- **Provenance and feedback** have no trait-level extension point. There is no way to distinguish tool-result facts (high trust) from assistant speculation (low trust) at the interface level, nor to provide feedback on whether a recalled memory was helpful.

Because these policies live outside the backend, adding a new backend (e.g., Postgres, a knowledge-graph hybrid) does not let you plug in a new retrieval or consolidation strategy; the runtime forces its own decay, filtering, and formatting on every backend.

### Proposed solution

Create a new `MemoryStrategy` trait in `zeroclaw-api` that owns the memory lifecycle policy. A default implementation, `DefaultMemoryStrategy`, wraps any `Arc<dyn Memory>` and contains the logic currently split across `DefaultMemoryLoader`, `consolidation::consolidate_turn`, and `hygiene::run_if_due`.

```rust
#[async_trait]
pub trait MemoryStrategy: Send + Sync {
    async fn load_context(&self, query: &str, opts: &ContextLoadOptions) -> Result<String>;
    async fn consolidate_turn(&self, turn: &ConversationTurn, provider: Option<&dyn ModelProvider>) -> Result<()>;
    async fn run_governance(&self) -> Result<GovernanceReport>;
    async fn feedback(&self, entry_id: &str, helpful: bool) -> Result<()>;
}
```

**Migration plan**
1. Define `MemoryStrategy` and `DefaultMemoryStrategy` in `zeroclaw-memory`.
2. Migrate `DefaultMemoryLoader::load_context` logic into `DefaultMemoryStrategy::load_context`.
3. Migrate `consolidation::consolidate_turn` logic into `DefaultMemoryStrategy::consolidate_turn`.
4. Migrate `hygiene::run_if_due` logic into `DefaultMemoryStrategy::run_governance`.
5. Update `Agent` to hold `memory_strategy: Arc<dyn MemoryStrategy>` instead of `memory_loader: Box<dyn MemoryLoader>`.
6. Unify consolidation trigger: call `memory_strategy.consolidate_turn()` inside `Agent::turn()` / `turn_streamed()` after the turn completes, removing the ad-hoc `tokio::spawn` calls in the gateway and channel orchestrator.
7. Deprecate `MemoryLoader` trait (or keep it as an optional override path).

This makes `Memory` a pure storage engine and `MemoryStrategy` the "brain" that can be swapped or extended (e.g., a `DreamingMemoryStrategy` for batch sleep-phase consolidation) without touching SQLite, Postgres, Qdrant, or Markdown backends.

### Non-goals / out of scope

- Changing the storage schema or `MemoryEntry` fields in this iteration; provenance tagging can follow once the strategy layer exists.
- UI or CLI changes beyond replacing the consolidation trigger path.
- Cross-backend consolidation (e.g., consolidating from SQLite into Qdrant) — the strategy wraps one `Memory` instance.

### Alternatives considered

1. **Add default methods to the existing `Memory` trait** (e.g., `Memory::load_context`, `Memory::consolidate_turn`). This keeps the trait surface smaller at the cost of forcing every backend to carry policy logic even when it only wants to be a dumb store. It also makes runtime downcasting/capability detection messier.
2. **Keep the status quo** — `MemoryLoader` for retrieval, standalone functions for consolidation and hygiene. This is what we have today and is the root cause of the duplication and path inconsistency.

### Acceptance criteria

- [ ] `MemoryStrategy` trait is defined in `zeroclaw-api` with the four lifecycle methods.
- [ ] `DefaultMemoryStrategy` is implemented in `zeroclaw-memory` and passes existing `MemoryLoader` + consolidation tests.
- [ ] `Agent::turn()` and `Agent::turn_streamed()` call `memory_strategy.consolidate_turn()` uniformly; gateway and channel orchestrator no longer spawn consolidation manually.
- [ ] `MemoryLoader` trait is deprecated with a forwarding note to `MemoryStrategy`.
- [ ] No regressions in `memory_loop_continuity` integration tests.

### Architecture impact

`memory/` (new trait + default impl), `runtime/` (Agent integration), `gateway/` + `channels/` (remove manual consolidation spawn), `docs/` (architecture note).

### Risk and rollback

**Risk**: `Agent` struct changes its memory-related field types, which affects any code constructing `Agent` directly (mostly internal).  
**Rollback**: Revert the Agent field change and restore the old `MemoryLoader` + manual spawn paths; the backend `Memory` trait itself is untouched, so no storage-layer rollback is needed.

### Breaking change?

No

### Data hygiene checks

- [x] I removed personal/sensitive data from examples, payloads, and logs.
- [x] I used neutral, project-focused wording and placeholders.
