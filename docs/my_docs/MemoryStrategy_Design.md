# MemoryStrategy Trait 设计方案

> 本方案基于社区 maintainer 反馈（#6850）调整，聚焦 Phase 1（Design/API PR），保持现有行为完全不变。`DefaultMemoryStrategy` 不复制任何逻辑，仅通过薄包装层委托调用现有实现。

---

## 1. 背景

### 1.1 当前问题

ZeroClaw 的 `Memory` trait（`crates/zeroclaw-api/src/memory_traits.rs`）目前仅提供底层存储原语（CRUD），而所有高阶记忆策略行为散落在 runtime 各处：

- **提取逻辑三处分裂**：`RetrievalPipeline` 仅做热缓存路由；`DefaultMemoryLoader`（`crates/zeroclaw-runtime/src/agent/memory_loader.rs`）执行 time-decay + 过滤 + `[Memory context]` 格式化；`build_context()`（`crates/zeroclaw-runtime/src/agent/loop_.rs:375-441`）又内联了一套几乎相同的逻辑。
- **巩固触发路径不一致**：`consolidation::consolidate_turn` 仅在 WebSocket gateway 和 channel orchestrator 中以 `tokio::spawn` 触发；`Agent::turn()`（library 路径）完全不触发。
- **治理机制无接口**：`hygiene::run_if_due` 是独立过程函数，不在任何 trait 上。

### 1.2 关联 Issue

本方案是 #5849（Dream Mode / 记忆巩固总体规划）的底层设计切片。

---

## 2. 目标

1. **解耦存储与策略**：`Memory` 保持纯 CRUD，`MemoryStrategy` 承载高阶记忆生命周期策略。
2. **消除重复逻辑**：将 `DefaultMemoryLoader`、`build_context`、`consolidation::consolidate_turn`、`hygiene::run_if_due` 统一收拢到 `DefaultMemoryStrategy`。
3. **统一调用入口**：`load_context`、`consolidate_turn`、`run_governance` 统一走 `MemoryStrategy` 接口。
4. **向后兼容**：现有 backend 零改动；旧接口保留，不废弃。

---

## 3. 接口与行为变更

### 3.1 新增 `MemoryStrategy` trait

定义位置：`crates/zeroclaw-api/src/memory_traits.rs`

```rust
/// Report produced by a governance pass.
#[derive(Debug, Clone, Default)]
pub struct GovernanceReport {
    pub archived_memory_files: u64,
    pub archived_session_files: u64,
    pub purged_memory_archives: u64,
    pub purged_session_archives: u64,
    pub pruned_conversation_rows: u64,
}

impl GovernanceReport {
    pub fn total_actions(&self) -> u64 {
        self.archived_memory_files
            + self.archived_session_files
            + self.purged_memory_archives
            + self.purged_session_archives
            + self.pruned_conversation_rows
    }
}

/// High-level memory lifecycle policy.
/// Implemented by strategy objects that wrap one or more `Memory` backends.
#[async_trait]
pub trait MemoryStrategy: Send + Sync {
    /// Load and format relevant memory context for a conversation turn.
    async fn load_context(
        &self,
        query: &str,
        session_id: Option<&str>,
    ) -> anyhow::Result<String>;

    /// Consolidate a conversation turn into long-term memory.
    async fn consolidate_turn(
        &self,
        user_message: &str,
        assistant_response: &str,
        provider: Option<&dyn ModelProvider>,
        model: Option<&str>,
        temperature: Option<f64>,
    ) -> anyhow::Result<()>;

    /// Run memory governance (cleanup, archiving, background consolidation).
    async fn run_governance(&self) -> anyhow::Result<GovernanceReport>;

    /// Provide feedback on whether a recalled entry was helpful.
    async fn feedback(&self, entry_id: &str, helpful: bool) -> anyhow::Result<()>;
}
```

### 3.2 新增 `DefaultMemoryStrategy`

定义位置：`crates/zeroclaw-memory/src/strategy.rs`（新建）

```rust
use std::sync::Arc;
use zeroclaw_api::memory_traits::{GovernanceReport, Memory, MemoryStrategy};
use zeroclaw_api::model_provider::ModelProvider;

pub struct DefaultMemoryStrategy {
    memory: Arc<dyn Memory>,
    limit: usize,
    min_relevance_score: f64,
    workspace_dir: std::path::PathBuf,
    // Throttle state for governance.
    last_governance_at: std::sync::Mutex<Option<std::time::Instant>>,
}

impl DefaultMemoryStrategy {
    pub fn new(
        memory: Arc<dyn Memory>,
        limit: usize,
        min_relevance_score: f64,
        workspace_dir: impl Into<std::path::PathBuf>,
    ) -> Self {
        Self {
            memory,
            limit,
            min_relevance_score,
            workspace_dir: workspace_dir.into(),
            last_governance_at: std::sync::Mutex::new(None),
        }
    }

    /// Convenience constructor matching DefaultMemoryLoader defaults.
    pub fn with_defaults(memory: Arc<dyn Memory>, workspace_dir: impl Into<std::path::PathBuf>) -> Self {
        Self::new(memory, 5, 0.4, workspace_dir)
    }
}

#[async_trait]
impl MemoryStrategy for DefaultMemoryStrategy {
    async fn load_context(
        &self,
        query: &str,
        session_id: Option<&str>,
    ) -> anyhow::Result<String> {
        let loader = DefaultMemoryLoader::new(self.limit, self.min_relevance_score);
        loader.load_context(self.memory.as_ref(), query, session_id).await
    }

    async fn consolidate_turn(
        &self,
        user_message: &str,
        assistant_response: &str,
        provider: Option<&dyn ModelProvider>,
        model: Option<&str>,
        temperature: Option<f64>,
    ) -> anyhow::Result<()> {
        let Some(provider) = provider else { return Ok(()); };
        let Some(model) = model else { return Ok(()); };
        zeroclaw_memory::consolidation::consolidate_turn(
            provider, model, temperature, self.memory.as_ref(), user_message, assistant_response,
        ).await
    }

    async fn run_governance(&self) -> anyhow::Result<GovernanceReport> {
        // Phase 1: 委托现有 hygiene 逻辑，GovernanceReport 留待 hygiene 模块暴露结构化报告后填充
        zeroclaw_memory::hygiene::run_if_due(&self.config, &self.workspace_dir,
        )?;
        Ok(GovernanceReport::default())
    }

    async fn feedback(
        &self,
        _entry_id: &str,
        _helpful: bool,
    ) -> anyhow::Result<()> {
        // Phase 1 为 no-op，预留接口
        Ok(())
    }
}
```

### 3.3 调用时序变更

**旧行为**：
- `Agent::turn()` -> `memory_loader.load_context()` -> LLM -> 返回
- gateway/channel -> `tokio::spawn(consolidation::consolidate_turn(...))`
- `lib.rs` 创建 memory 时 -> `hygiene::run_if_due()`

**新行为**：
- `Agent::turn()` -> `memory_strategy.load_context()` -> LLM -> 返回
- gateway/channel -> `tokio::spawn(memory_strategy.consolidate_turn(...))`
- `Agent` 初始化/定时任务 -> `memory_strategy.run_governance()`

> **注意**：Phase 1 不改变 consolidation 的触发路径（gateway/channel 仍各自 spawn，`Agent::turn()` 仍不触发），仅把被调用的目标从独立函数改为 `memory_strategy` 的方法。

---

## 4. 实现思路

### 4.1 `zeroclaw-api`

在 `memory_traits.rs` 中定义 `MemoryStrategy` trait 和 `GovernanceReport`。不修改现有 `Memory` trait。

### 4.2 `zeroclaw-memory`

- **新建 `strategy.rs`**：实现 `DefaultMemoryStrategy`。
  - `load_context`：委托调用 `DefaultMemoryLoader`。
  - `consolidate_turn`：委托调用 `consolidation::consolidate_turn`。
  - `run_governance`：委托调用 `hygiene::run_if_due`。
- **`lib.rs`**：导出 `strategy` 模块；删除 `hygiene::run_if_due` 的硬编码调用（改为由 Agent 侧触发）。
- **保留旧模块**：`memory_loader.rs`、`consolidation.rs`、`hygiene.rs` 完全不动，不做废弃标记。`strategy.rs` 是纯粹的新增文件。

### 4.3 `zeroclaw-runtime`

- **`Agent`**：新增 `memory_strategy: Arc<dyn MemoryStrategy>` 字段；`turn()` / `turn_streamed()` 调用 `memory_strategy.load_context()`。
- **`loop_.rs`**：`build_context()` 改为调用 `memory_strategy.load_context()`。

### 4.4 `zeroclaw-gateway` / `zeroclaw-channels`

- 将手动 `tokio::spawn(consolidation::consolidate_turn(...))` 改为 `tokio::spawn(memory_strategy.consolidate_turn(...))`。
- 触发路径不变，仅替换被调用对象。

---

## 5. 关键变更点

| 文件 | 变更 | 备注 |
|------|------|------|
| `crates/zeroclaw-api/src/memory_traits.rs` | **新增** `MemoryStrategy` trait、`GovernanceReport` | 无破坏性变更 |
| `crates/zeroclaw-memory/src/strategy.rs` | **新建** `DefaultMemoryStrategy` | 收拢策略逻辑 |
| `crates/zeroclaw-memory/src/lib.rs` | 导出 `strategy` 模块；**删除** `hygiene::run_if_due` 硬编码调用 | governance 改由 Agent 侧触发 |
| `crates/zeroclaw-runtime/src/agent/agent.rs` | `Agent` 新增 `memory_strategy`；`turn()` / `turn_streamed()` 调用 `memory_strategy.load_context()` | 切调用点 |
| `crates/zeroclaw-runtime/src/agent/loop_.rs` | `build_context()` 改为调用 `memory_strategy.load_context()` | 切调用点 |
| `crates/zeroclaw-gateway/src/ws.rs` | spawn consolidation 改为调用 `memory_strategy.consolidate_turn()` | 切调用点，触发路径不变 |
| `crates/zeroclaw-channels/src/orchestrator/mod.rs` | spawn consolidation 改为调用 `memory_strategy.consolidate_turn()` | 切调用点，触发路径不变 |
| `tests/` | **新增** `DefaultMemoryStrategy` 与现有代码的**等价性测试** | 验证行为一致 |

---

## 6. 测试策略

**核心原则**：`DefaultMemoryStrategy` 作为薄包装层，其输出必须与委托的现有实现逐字节等价。

- **Context 等价性测试**：对同一组 `MemoryEntry`，`DefaultMemoryStrategy::load_context`（内部委托 `DefaultMemoryLoader`）的输出应与直接调用 `DefaultMemoryLoader::load_context` 完全一致。验证委托链路无信息丢失。
- **Consolidation 等价性测试**：对同一组输入，`DefaultMemoryStrategy::consolidate_turn`（内部委托 `consolidation::consolidate_turn`）写入 memory 的条目应与直接调用原函数一致。
- **Governance 等价性测试**：`DefaultMemoryStrategy::run_governance`（内部委托 `hygiene::run_if_due`）的行为应与直接调用原函数一致。
- **端到端黑盒测试**：复用现有 `memory_loop_continuity` 测试，确保无回归。

---

## 7. 明确不做的事

| 项 | 原因 |
|----|------|
| 不改变过滤行为 | `load_context` 保持 `DefaultMemoryLoader` 的过滤逻辑，不新增也不删减 |
| 不废弃 `MemoryLoader` | 保持完全向后兼容，废弃标记是后续阶段的事 |
| 不统一 consolidation 触发路径 | gateway/channel 仍各自 `tokio::spawn`；`Agent::turn()` 不新增 consolidation 调用 |
| 不碰 provenance / feedback / cross-backend / dreaming | maintainer 明确要求作为 follow-up |

---

## 8. 风险与回滚

| 风险 | 缓解措施 |
|------|----------|
| `Agent` 字段变更影响内部构造点 | 所有构造走 builder/factory，集中修改一处 |
| gateway/channel 调用路径遗漏 | 全局 grep `consolidate_turn` 确保所有调用点已迁移到 `memory_strategy` |
| `DefaultMemoryStrategy` 行为与原代码不一致 | 等价性测试作为回归防护 |
| trait 变更后下游 crate 编译失败 | `MemoryStrategy` 是新 trait，不影响现有 `Memory` impl |

**回滚方案**：恢复 `Agent` 字段为 `memory_loader`；在 gateway/channel 中恢复直接调用 `consolidation::consolidate_turn`；恢复 `lib.rs` 中的 `run_if_due` 调用。`Memory` trait 本身未被修改，存储层无需回滚。

---

## 9. 维护者反馈对应

| 维护者原话 | 本方案对应 |
|-----------|-----------|
| *"first PR to land as one broad API/runtime/gateway/channels refactor all at once"* → 不希望 | 本方案限定为策略层抽取 + 调用点迁移，无新行为变更 |
| *"first slice would be a design/API PR... DefaultMemoryStrategy preserving current behavior"* | 完整对应：定义 trait + DefaultMemoryStrategy，所有调用点切到 strategy，行为逐字节不变 |
| *"follow with smaller implementation PRs that unify the entry points"* | consolidation 触发路径的统一、废弃标记、build_context 删除等，全部留到后续阶段 |
| *"keep provenance tagging, feedback, and cross-backend strategy behavior as follow-ups"* | 明确列入"不做的事"清单 |

---

*设计状态：草案（待 Phase 1 PR Review）*
*关联 Issue：#6850、#5849*
*关联报告：`memory-interface-extensibility-report.md`*
