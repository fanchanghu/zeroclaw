# ZeroClaw 记忆系统设计文档

## 一、设计哲学

ZeroClaw 的记忆系统遵循 **"分层隔离、显式优于隐式"** 的设计原则：

- **trait 驱动**：统一接口，多后端实现
- **类别分层**：Conversation（短期）与 Core（长期）明确区分
- **显式控制**：长期记忆需显式写入，避免自动保存污染长期存储

---

## 二、核心架构

### 2.1 模块结构

```
src/memory/
├── mod.rs           # 工厂入口
├── traits.rs        # Memory trait 定义
├── sqlite.rs        # SQLite 后端（默认）
├── lucid.rs         # Lucid 分层后端
├── qdrant.rs        # Qdrant 向量后端
├── markdown.rs      # Markdown 文件后端
├── embeddings.rs    # 向量化
├── retrieval.rs     # 检索管道
├── hygiene.rs       # 定期清理
├── snapshot.rs      # 快照导出/恢复
├── decay.rs         # 时间衰减
├── importance.rs    # 重要性评分
└── policy.rs        # 策略执行
```

### 2.2 工厂模式

```rust
// src/memory/mod.rs
pub fn create_memory(
    config: &MemoryConfig,
    workspace_dir: &Path,
    api_key: Option<&str>,
) -> anyhow::Result<Box<dyn Memory>> {
    match classify_memory_backend(backend_name) {
        MemoryBackendKind::Sqlite => Ok(Box::new(sqlite_builder()?)),
        MemoryBackendKind::Lucid => Ok(Box::new(LucidMemory::new(workspace_dir, local))),
        MemoryBackendKind::Postgres => postgres_builder(),
        MemoryBackendKind::Markdown => Ok(Box::new(MarkdownMemory::new(workspace_dir))),
        MemoryBackendKind::None => Ok(Box::new(NoneMemory::new())),
        _ => fallback_to_markdown(),
    }
}
```

---

## 三、核心接口设计

### 3.1 Memory Trait

```rust
// src/memory/traits.rs
#[async_trait]
pub trait Memory: Send + Sync {
    fn name(&self) -> &str;

    /// 存储记忆
    async fn store(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
    ) -> anyhow::Result<()>;

    /// 检索记忆（核心方法）
    async fn recall(
        &self,
        query: &str,
        limit: usize,
        session_id: Option<&str>,
        since: Option<&str>,
        until: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>>;

    async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>>;
    async fn list(&self, category: Option<&MemoryCategory>, session_id: Option<&str>) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn forget(&self, key: &str) -> anyhow::Result<bool>;
}
```

### 3.2 记忆类别设计

```rust
// src/memory/traits.rs
pub enum MemoryCategory {
    /// 长期事实、偏好、关键决策（永不衰减）
    Core,
    /// 每日会话日志（14天保留）
    Daily,
    /// 对话上下文（7天保留，自动保存）
    Conversation,
    /// 用户自定义类别
    Custom(String),
}
```

**关键设计**：类别决定生命周期策略
- **Core**：`evergreen`（`decay.rs` 中跳过衰减），保留 365 天
- **Conversation**：自动保存用户消息，但短期保留
- **Daily**：特定渠道（如 Discord）的会话日志

---

## 四、存储触发机制

### 4.1 自动保存（仅 Conversation）

```rust
// src/agent/loop_.rs
const AUTOSAVE_MIN_MESSAGE_CHARS: usize = 20;

// 自动保存用户消息到 Conversation（仅当长度≥20字符）
if config.memory.auto_save
    && effective_msg.chars().count() >= AUTOSAVE_MIN_MESSAGE_CHARS
    && !memory::should_skip_autosave_content(&effective_msg)
{
    let user_key = format!("user_msg_{}", Uuid::new_v4());
    let _ = mem.store(
        &user_key,
        &effective_msg,
        MemoryCategory::Conversation,  // ← 固定为 Conversation
        memory_session_id.as_deref(),
    ).await;
}
```

### 4.2 显式工具调用（Core/Daily/Custom）

```rust
// src/tools/memory_store.rs
async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
    let category = match args.get("category").and_then(|v| v.as_str()) {
        Some("core") | None => MemoryCategory::Core,
        Some("daily") => MemoryCategory::Daily,
        Some("conversation") => MemoryCategory::Conversation,
        Some(other) => MemoryCategory::Custom(other.to_string()),
    };

    self.memory.store(key, content, category, None).await
}
```

**设计意图**：长期记忆（Core）需显式声明，避免自动保存的噪声污染。

---

## 五、检索与上下文注入

### 5.1 自动上下文构建

```rust
// src/agent/loop_.rs
async fn build_context(
    mem: &dyn Memory,
    user_msg: &str,
    min_relevance_score: f64,
    session_id: Option<&str>,
) -> String {
    // 检索相关记忆（默认返回5条）
    if let Ok(mut entries) = mem.recall(user_msg, 5, session_id, None, None).await {
        // 应用时间衰减（Core 除外）
        decay::apply_time_decay(&mut entries, decay::DEFAULT_HALF_LIFE_DAYS);

        // 过滤低相关度
        let relevant: Vec<_> = entries.iter()
            .filter(|e| e.score.map_or(true, |s| s >= min_relevance_score))
            .filter(|e| !memory::is_assistant_autosave_key(&e.key))  // 过滤 assistant_resp
            .filter(|e| !e.content.contains("<tool_result"))         // 过滤工具结果
            .collect();
    }
}
```

### 5.2 工具显式召回

```rust
// src/tools/memory_recall.rs
async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
    let entries = self.memory.recall(query, limit, None, since, until).await?;
    // 返回格式化结果给 LLM
}
```

---

## 六、生命周期管理

### 6.1 卫生清理（Hygiene）

```rust
// src/memory/hygiene.rs
pub fn run_if_due(config: &MemoryConfig, workspace_dir: &Path) -> anyhow::Result<()> {
    // 按类别清理过期记忆
    purge_by_category(&MemoryCategory::Conversation, config.conversation_retention_days)?;
    purge_by_category(&MemoryCategory::Daily, config.daily_retention_days)?;
    // Core 不自动清理
}
```

### 6.2 快照与冷启动恢复

```rust
// src/memory/mod.rs - create_memory 中
// 1. 卫生清理（定期执行）
hygiene::run_if_due(config, workspace_dir)?;

// 2. 快照导出（如启用）
if config.snapshot_on_hygiene {
    snapshot::export_snapshot(workspace_dir)?;
}

// 3. 冷启动恢复（ brain.db 缺失时从快照恢复）
if config.auto_hydrate && snapshot::should_hydrate(workspace_dir) {
    snapshot::hydrate_from_snapshot(workspace_dir)?;
}
```

---

## 七、Embedding 与向量检索

### 7.1 路由配置

```rust
// 支持 hint: 前缀的路由配置
let resolved = resolve_embedding_config(&cfg, &routes, api_key);

// 优先级：
// 1. 路由配置的 api_key
// 2. 提供商专用环境变量（OPENAI_API_KEY/COHERE_API_KEY）
// 3. 调用者传入的默认 key（防止 chat key 泄露到 embedding）
```

---

## 八、关键设计决策

| 决策 | 说明 |
|------|------|
| **Conversation 自动保存** | 用户消息自动入库，但短期保留（7天）|
| **Core 显式写入** | 长期记忆需通过 `memory_store` 工具显式创建 |
| **Core 永不衰减** | `decay.rs` 中明确跳过 Core 类别 |
| **Embedding key 隔离** | 防止 chat provider key 泄露到 embedding 端点 |
| **冷启动恢复** | `MEMORY_SNAPSHOT.md` 导出 Core 记忆，支持跨实例恢复 |

---

## 九、使用模式总结

```
┌─────────────────────────────────────────────────────────────┐
│                     记忆写入模式                              │
├─────────────────────────────────────────────────────────────┤
│  自动：用户消息 → Conversation（临时）                        │
│  显式：memory_store 工具 → Core/Daily/Custom（长期）          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     记忆读取模式                              │
├─────────────────────────────────────────────────────────────┤
│  自动：build_context() 检索相关记忆注入上下文                  │
│  显式：memory_recall 工具按需查询                             │
└─────────────────────────────────────────────────────────────┘
```

这种设计确保了**短期记忆的自动化**与**长期记忆的显式控制**之间的平衡。

---

## 十、程序记忆：Skill 系统

### 10.1 核心概念

ZeroClaw 的 **Skill 系统** 是一种特殊的**程序记忆（Procedural Memory）**——它将成功的多步骤任务执行沉淀为可复用的能力模块。

| 记忆类型 | 存储内容 | 典型示例 |
|---------|---------|---------|
| **陈述记忆（Declarative）** | 事实、信息、历史记录 | "用户偏好深色模式" |
| **程序记忆（Procedural）** | 如何执行任务的能力 | "构建项目的标准流程" |

Skill 系统的本质是：**从执行中学习，将经验固化为可复用的程序**。

### 10.2 自动生成机制

当 agent 完成一个包含 2+ 工具调用的多步骤任务后，`SkillCreator` 自动将其转换为可复用的 skill：

```rust
// src/skills/creator.rs
pub async fn create_from_execution(
    &self,
    task_description: &str,
    tool_calls: &[ToolCallRecord],
    embedding_provider: Option<&dyn EmbeddingProvider>,
) -> Result<Option<String>>
```

**触发条件**：
- 任务成功完成
- 涉及 ≥2 个工具调用（排除简单单步操作）
- 未检测到相似 skill 存在（通过 embedding 相似度去重，默认阈值 0.85）

**生成内容**：
```toml
# ~/.zeroclaw/workspace/skills/build-and-test/SKILL.toml
[skill]
name = "build-and-test"
description = "Auto-generated: Build and test the project"
version = "0.1.0"
author = "zeroclaw-auto"
tags = ["auto-generated"]

[[tools]]
name = "shell"
description = "Tool used in task: shell"
kind = "shell"
command = "cargo build"

[[tools]]
name = "shell"
description = "Tool used in task: shell"
kind = "shell"
command = "cargo test"
```

### 10.3 Skill 与记忆系统的协同

```
┌─────────────────────────────────────────────────────────────────┐
│                     程序记忆生命周期                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   任务执行 ──→ 成功检测 ──→ SkillCreator ──→ 去重检查            │
│      ↑                                          │               │
│      └──────── 复用已有 Skill ←─────────────────┘               │
│                          ↓                                      │
│                    生成 SKILL.toml                              │
│                          ↓                                      │
│                    加载到 Prompt                                  │
│                          ↓                                      │
│                    后续任务复用                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**与声明记忆的交互**：
- **Skill 使用记录**：作为 `Core` 记忆存储（"我之前用过 build-and-test skill"）
- **Skill 改进反馈**：通过 `SkillImprover` 原子更新现有 skill，记录改进原因

### 10.4 生命周期管理

| 维度 | 策略 | 说明 |
|-----|------|------|
| **数量限制** | LRU 淘汰 | 默认最多保留 500 个自动生成 skill |
| **去重机制** | Embedding 相似度 | 新任务描述与现有 skill 描述相似度 >0.85 时跳过 |
| **身份标识** | 元数据标记 | `author = "zeroclaw-auto"`, `tags = ["auto-generated"]` |
| **改进追踪** | 原子更新 | `SkillImprover` 支持带冷却期的增量改进 |

### 10.5 与记忆系统的统一视角

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZeroClaw 记忆层次架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  工作记忆（Working Memory）                              │   │
│  │  ─────────────────────────                               │   │
│  │  当前对话上下文、系统提示、已加载 Skill 指令               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  短期记忆（Short-term）                                  │   │
│  │  ─────────────────────                                   │   │
│  │  Conversation：7天自动衰减                               │   │
│  │  Daily：14天会话日志                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  长期记忆（Long-term）                                   │   │
│  │  ────────────────────                                    │   │
│  │  Core：显式存储的关键事实（365天）                        │   │
│  │  Skill：程序记忆，可复用的任务能力（LRU管理）              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.6 使用模式

```
┌─────────────────────────────────────────────────────────────┐
│                     Skill 写入模式                            │
├─────────────────────────────────────────────────────────────┤
│  自动：多步任务成功 → SkillCreator → SKILL.toml              │
│  显式：用户/开发者手动创建 skill 目录                         │
│  安装：zeroclaw skills install <source>                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Skill 读取模式                            │
├─────────────────────────────────────────────────────────────┤
│  自动：load_skills() 加载所有 skill 注入系统提示              │
│  按需：Compact 模式下通过 read_skill(name) 动态加载           │
│  显式：skill 工具直接调用（shell/http/script 类型）           │
└─────────────────────────────────────────────────────────────┘
```

**设计意图**：Skill 作为程序记忆，填补了"陈述记忆只能存储信息"与"agent 需要复用能力"之间的 gap，实现了**从经验到能力的自动沉淀**。
