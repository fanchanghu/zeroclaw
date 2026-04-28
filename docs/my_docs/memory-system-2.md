# ZeroClaw 记忆系统

ZeroClaw的记忆机制是一个**以文件为核心、多层次、可搜索、高度可配置的持久化系统**。它旨在解决大语言模型原生记忆的不可靠与临时性问题，为智能体提供稳定、可检索的长期记忆能力。

## 📁 核心设计：文件即记忆

ZeroClaw最根本的原则是，记忆并非存储在模型参数或运行时内存中，而是**存储在智能体工作空间的持久化存储中**。这意味着：

*   **唯一事实来源**：模型只能"记住"已写入持久存储的内容。
*   **持久性与可读性**：记忆以标准格式存在，易于用户查看、编辑和管理。
*   **分层结构**：提供多个核心记忆类别：
    *   **核心记忆 (`core`)**：用于存储长期事实、用户偏好、关键决策等持久性信息。
    *   **每日日志 (`daily`)**：用于记录每日运行上下文、笔记和对话。
    *   **对话上下文 (`conversation`)**：存储临时的对话内容，支持会话隔离。
    *   **自定义类别 (`custom`)**：用户定义的任意类别，用于特定场景。

## 🏗️ 架构概览

ZeroClaw的记忆系统采用模块化设计，基于统一的`Memory` trait接口，支持多种后端存储：

```rust
pub trait Memory: Send + Sync {
    fn name(&self) -> &str;
    async fn store(&self, key: &str, content: &str, category: MemoryCategory, session_id: Option<&str>) -> anyhow::Result<()>;
    async fn recall(&self, query: &str, limit: usize, session_id: Option<&str>) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>>;
    async fn list(&self, category: Option<&MemoryCategory>, session_id: Option<&str>) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn forget(&self, key: &str) -> anyhow::Result<bool>;
    async fn count(&self) -> anyhow::Result<usize>;
    async fn health_check(&self) -> bool;
}
```

## 🔧 支持的后端

ZeroClaw提供多种记忆后端，满足不同场景需求：

### 1. **SQLite (推荐)**
   - **标签**: `sqlite`
   - **特点**: 本地SQLite数据库，支持向量搜索和混合检索
   - **优势**: 快速、轻量、支持混合搜索（向量+关键词）、嵌入式向量计算
   - **适用场景**: 大多数本地部署场景，需要语义搜索能力

### 2. **Lucid Memory Bridge**
   - **标签**: `lucid`
   - **特点**: 与本地lucid-memory CLI同步，保持SQLite回退
   - **优势**: 云同步能力，本地回退保障
   - **适用场景**: 需要云同步的多设备场景

### 3. **Markdown Files**
   - **标签**: `markdown`
   - **特点**: 纯Markdown文件存储，人类可读
   - **优势**: 无依赖、易于版本控制、直接可编辑
   - **适用场景**: 简单部署、需要人工审核记忆内容

### 4. **PostgreSQL**
   - **标签**: `postgres`
   - **特点**: 远程持久存储，通过`[storage.provider.config]`配置
   - **优势**: 企业级可靠性、多客户端访问
   - **适用场景**: 团队协作、高可用部署

### 5. **Qdrant**
   - **标签**: `qdrant`
   - **特点**: 向量数据库专为语义搜索设计
   - **优势**: 高性能向量检索、可扩展性
   - **适用场景**: 大规模记忆库、专业向量搜索需求

### 6. **None**
   - **标签**: `none`
   - **特点**: 禁用持久记忆
   - **优势**: 无存储开销
   - **适用场景**: 临时会话、测试环境

## ✍️ 记忆写入机制

系统通过多种方式确保信息被可靠存储：

### 1. **显式指令**
   当用户或系统指令（如"记住这个"）出现时，模型会通过`memory_store`工具写入相应文件。

### 2. **自动保存**
   在`[memory]`配置中启用`auto_save = true`时，系统会自动保存用户消息到记忆：
   ```toml
   [memory]
   auto_save = true
   ```

### 3. **工具调用**
   智能体可以通过以下工具直接操作记忆：
   - `memory_store` - 存储记忆条目
   - `memory_recall` - 检索相关记忆
   - `memory_get` - 获取特定键的记忆
   - `memory_forget` - 删除记忆
   - `memory_list` - 列出记忆条目
   - `memory_count` - 统计记忆数量

### 4. **会话隔离**
   支持通过`session_id`参数实现记忆的会话隔离，确保不同会话间的记忆不混淆：
   ```rust
   mem.store("preference", "likes Rust", MemoryCategory::Core, Some("session-123")).await?;
   ```

## 🔍 智能检索：混合搜索系统

ZeroClaw不仅能存储，还能高效检索。它在记忆上构建了智能搜索系统：

### **向量搜索 (语义检索)**
   - **嵌入提供商**: 支持多种嵌入模型，按优先级自动选择（本地、OpenAI、Gemini），也可手动配置
   - **批处理**: 支持批处理以优化大型索引的成本和效率
   - **缓存**: 内置**嵌入缓存**避免重复计算

### **关键词搜索 (BM25检索)**
   - **FTS5全文搜索**: 使用SQLite的FTS5引擎进行快速关键词匹配
   - **精确匹配**: 擅长匹配ID、代码符号、专有名词等

### **混合搜索**
   这是其检索能力的亮点。它结合了：
   - **向量相似度**（擅长语义匹配，如"Mac Studio网关主机"与"运行网关的机器"）
   - **BM25关键词匹配**（擅长精确匹配ID、代码符号等）

   通过权重配置融合结果，实现更精准的搜索：
   ```toml
   [memory]
   vector_weight = 0.7    # 向量相似度权重
   keyword_weight = 0.3   # 关键词匹配权重
   ```

## 🧹 记忆卫生与快照

### **自动卫生管理**
ZeroClaw定期执行记忆卫生任务，包括：
- **归档旧文件**: 自动将旧的每日日志和会话文件移动到archive目录
- **清理过期数据**: 根据配置的保留策略删除过期记忆
- **数据库维护**: SQLite数据库的优化和清理

配置示例：
```toml
[memory]
hygiene_enabled = true
archive_after_days = 30      # 30天后归档
purge_after_days = 90        # 90天后清理
conversation_retention_days = 7  # 对话记忆保留7天
```

### **原子灵魂快照**
ZeroClaw提供独特的"灵魂快照"功能：

#### **快照导出**
将核心记忆导出为人类可读的Markdown文件`MEMORY_SNAPSHOT.md`：
```bash
# 自动在卫生任务中导出
[memory]
snapshot_enabled = true
snapshot_on_hygiene = true
```

#### **自动水合**
当数据库丢失但快照存在时，自动从快照恢复记忆：
```toml
[memory]
auto_hydrate = true  # 启用自动水合
```

**水合场景**：
1. `brain.db`不存在或为空
2. `MEMORY_SNAPSHOT.md`存在
3. 系统自动从快照恢复核心记忆

## 💾 响应缓存

为避免在重复提示上浪费token，ZeroClaw提供可选的响应缓存：

```toml
[memory]
response_cache_enabled = true
response_cache_ttl_minutes = 60      # 缓存有效期60分钟
response_cache_max_entries = 1000    # 最大缓存条目数
```

**缓存键生成**：基于`(模型, 系统提示哈希, 用户提示)`的SHA-256哈希

## ⚙️ 配置详解

### 基础配置
```toml
[memory]
backend = "sqlite"                    # 后端类型
auto_save = true                      # 自动保存用户消息

# 嵌入配置
embedding_provider = "openai"         # 嵌入提供商
embedding_model = "text-embedding-3-small"  # 嵌入模型
embedding_dimensions = 1536           # 向量维度

# 搜索权重
vector_weight = 0.7                   # 向量相似度权重
keyword_weight = 0.3                  # 关键词权重

# 缓存配置
embedding_cache_size = 10000          # 嵌入缓存大小
```

### 嵌入路由
通过路由提示实现灵活的嵌入配置：
```toml
[memory]
embedding_model = "hint:semantic"    # 使用路由提示

[[embedding_routes]]
hint = "semantic"
provider = "openai"
model = "text-embedding-3-small"
dimensions = 1536
api_key = "sk-..."                    # 可选的API密钥覆盖
```

### Qdrant专用配置
```toml
[memory]
backend = "qdrant"

[memory.qdrant]
url = "http://localhost:6333"         # Qdrant服务器URL
collection = "zeroclaw_memories"      # 集合名称
api_key = ""                          # 可选的API密钥
```

## 🛠️ CLI工具

ZeroClaw提供完整的记忆管理CLI：

```bash
# 列出记忆条目
zeroclaw memory list
zeroclaw memory list --category core --limit 10
zeroclaw memory list --session session-123

# 获取特定记忆
zeroclaw memory get "user_preference_lang"

# 显示统计信息
zeroclaw memory stats

# 清理记忆
zeroclaw memory clear --category conversation --yes
zeroclaw memory clear --key "temp_note" --yes
```

## 🔄 迁移与互操作性

### 后端迁移
支持在不同后端间迁移记忆数据：
```bash
# 当前使用markdown，迁移到sqlite
zeroclaw config set memory.backend sqlite
# 系统会自动处理迁移
```

### 存储提供商覆盖
可通过存储提供商配置覆盖记忆后端：
```toml
[storage.provider.config]
provider = "postgres"          # 覆盖memory.backend设置
db_url = "postgresql://..."
schema = "public"
table = "memories"
```

## 🛡️ 安全特性

### 1. **会话隔离**
   记忆条目可关联到特定会话，防止跨会话信息泄露。

### 2. **自动保存过滤**
   忽略旧的`assistant_resp*`自动保存键，防止模型生成的摘要被误认为事实。

### 3. **输入验证**
   所有记忆操作都经过严格的输入验证和清理。

### 4. **加密存储**
   敏感配置（如API密钥）支持加密存储。

## 📊 性能优化

### SQLite优化
```rust
PRAGMA journal_mode = WAL;      # 写前日志，支持并发读写
PRAGMA synchronous = NORMAL;    # 平衡性能与耐久性
PRAGMA mmap_size = 8388608;     # 8MB内存映射
PRAGMA cache_size = -2000;      # 2MB缓存
PRAGMA temp_store = MEMORY;     # 临时表使用内存
```

### 批量操作
- 嵌入批处理减少API调用
- 缓存最近使用的嵌入向量
- 惰性初始化减少启动时间

## 🔍 故障排除

### 常见问题

1. **记忆检索不准确**
   ```bash
   # 检查嵌入配置
   zeroclaw doctor --check memory

   # 重建搜索索引
   zeroclaw memory reindex
   ```

2. **存储空间不足**
   ```bash
   # 清理旧记忆
   zeroclaw memory clear --category conversation --yes

   # 调整保留策略
   zeroclaw config set memory.archive_after_days 14
   ```

3. **性能问题**
   ```bash
   # 查看记忆统计
   zeroclaw memory stats

   # 优化数据库
   zeroclaw memory optimize
   ```

### 调试命令
```bash
# 检查记忆系统健康状态
zeroclaw doctor --verbose

# 查看详细日志
RUST_LOG=zeroclaw::memory=debug zeroclaw memory list

# 导出诊断信息
zeroclaw doctor export --output memory-diagnostics.json
```

## 🚀 最佳实践

### 1. **选择合适的后端**
   - **本地开发**: 使用`sqlite`或`markdown`
   - **生产环境**: 使用`sqlite`（单实例）或`postgres`（多实例）
   - **向量搜索需求**: 使用`qdrant`
   - **云同步需求**: 使用`lucid`

### 2. **优化搜索权重**
   根据使用场景调整向量和关键词权重：
   ```toml
   # 代码相关场景 - 侧重关键词匹配
   vector_weight = 0.3
   keyword_weight = 0.7

   # 语义理解场景 - 侧重向量匹配
   vector_weight = 0.8
   keyword_weight = 0.2
   ```

### 3. **合理使用记忆类别**
   - `core`: 用户偏好、重要事实、长期决策
   - `daily`: 日常日志、临时笔记
   - `conversation`: 对话上下文（设置合理保留时间）
   - `custom`: 项目特定分类

### 4. **启用快照功能**
   始终启用快照，确保核心记忆可恢复：
   ```toml
   [memory]
   snapshot_enabled = true
   auto_hydrate = true
   ```

### 5. **监控记忆使用**
   定期检查记忆统计：
   ```bash
   # 每周检查
   zeroclaw memory stats

   # 查看类别分布
   zeroclaw memory list --category core | wc -l
   ```

## 📚 相关文档

- [配置参考](config-reference.md) - 完整的记忆配置选项
- [CLI命令参考](cli/commands-reference.md) - 记忆管理命令
- [代理循环](agent-loop-run.md) - 记忆在代理循环中的使用
- [工具参考](../tools-reference.md) - 记忆相关工具
- [故障排除](../ops/troubleshooting.md) - 常见问题解决

---

ZeroClaw的记忆系统设计遵循"实用主义"哲学，在提供强大功能的同时保持简单可靠。通过模块化架构和智能检索能力，它为AI智能体提供了稳定、高效的记忆基础设施，是构建可靠自主代理的关键组件。
