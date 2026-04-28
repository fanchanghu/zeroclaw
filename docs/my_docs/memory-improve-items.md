# 记忆系统改进项

> 记录 ZeroClaw 记忆子系统在设计层面的待改进问题与方向性思考。

---

## 1. Assistant 回复的保存策略：从"完全不保存"到"批判性记忆"

### 当前设计

代码中仅自动保存 `user_msg` 到 `MemoryCategory::Conversation`，助手回复（`assistant_resp`）被明确排除：

- `is_assistant_autosave_key()` 将 `assistant_resp` 标记为 **untrusted context**
- 测试断言 `assistant_resp should not be auto-saved anymore`
- 理由：防止模型生成的幻觉/推断被重新注入上下文后自我放大

### 问题分析

#### 1.1 人格化断裂

Agent 只记得用户说过什么，不记得自己回应过什么。当用户说"你刚才说…"时，Agent 只能依赖用户的转述来间接推断，无法建立真正的对话连续性。

#### 1.2 丢失高价值上下文

助手回复中不全是幻觉，还包含大量对后续交互至关重要的信息：

- **工具调用结果**："我已经帮你把文件写好了"——如果不记得，用户说"谢谢"时 Agent 不知道在谢什么
- **对用户的承诺**："我下次会提醒你"——不保存就变成了空头支票
- **多步推理的中间结论**：后续 turn 可能引用前面的推理步骤
- **用户的反馈锚点**："你刚才说得对/不对"——如果不知道"刚才说了什么"，反馈就悬空

#### 1.3 过度简化

系统已经有了精细的信任分层（Core 0.7 / Daily 0.3 / Conversation 0.2）和冲突检测机制，本可以对不同可信度内容差异化处理。但对 assistant 内容采取绝对禁止，相当于"已经有了防盗门，还要把窗户封死"。

> **不相信 memory，和假装 memory 不存在，是两回事。**

### 建议改进方向

不是"保存所有助手回复"，而是**区分 assistant 回复中的不同成分**，引入 **provenance（来源标注）**：

| 内容类型 | 可信度 | 建议处理方式 |
|---------|--------|------------|
| 用户明确陈述的事实 | 高 | 已有：用户消息保存后进入 consolidation |
| 工具返回的客观结果 | 高 | **应新增保存**，这是 action 的客观痕迹 |
| 助手对用户的承诺/约定 | 中 | **应新增保存**，否则无法履约和追踪 |
| 助手的推理/分析过程 | 中-低 | 可选保存，短期 Conversation，快速衰减 |
| 助手无依据的推断/猜测 | 低 | 不保存，或明确标记为 `speculative` |

召回阶段根据 provenance 选择性采信，而非全盘信任或全盘否定。冲突检测模块也可以据此标记过时的 assistant 推断为 `superseded`，就像当前对 Core 记忆已经做的那样。

---

*记录时间：2026-04-29*
*关联代码：`crates/zeroclaw-memory/src/lib.rs:is_assistant_autosave_key`、`crates/zeroclaw-runtime/src/agent/loop_.rs` auto-save 逻辑、各 memory backend 的 `store` 调用点*
