# ZeroClaw Main 函数流程与 CLI 命令详解

本文档详细介绍 ZeroClaw 的 `main` 函数执行流程以及所有 CLI 命令的功能说明。

## 目录

- [Main 函数执行流程](#main-函数执行流程)
- [CLI 命令概览](#cli-命令概览)
- [命令详细说明](#命令详细说明)
  - [初始化与配置](#初始化与配置)
  - [运行时模式](#运行时模式)
  - [系统管理](#系统管理)
  - [安全与监控](#安全与监控)
  - [扩展功能](#扩展功能)

---

## Main 函数执行流程

`main` 函数是 ZeroClaw 的入口点，负责初始化系统、解析命令行参数并分发执行。以下是详细的执行流程：

### 1. 初始化阶段

```rust
// 安装 Rustls TLS 的默认加密提供者
if let Err(e) = rustls::crypto::ring::default_provider().install_default() {
    eprintln!("Warning: Failed to install default crypto provider: {e:?}");
}
```

- 安装默认的加密提供者，防止出现 "could not automatically determine the process-level CryptoProvider" 错误

### 2. 参数解析

```rust
let cli = Cli::parse();
```

- 使用 `clap` 解析命令行参数
- 定义了全局选项 `--config-dir` 用于指定配置目录

### 3. 配置目录处理

```rust
if let Some(config_dir) = &cli.config_dir {
    if config_dir.trim().is_empty() {
        bail!("--config-dir cannot be empty");
    }
    std::env::set_var("ZEROCLAW_CONFIG_DIR", config_dir);
}
```

- 如果指定了 `--config-dir`，设置环境变量 `ZEROCLAW_CONFIG_DIR`

### 4. 补全脚本生成（特殊处理）

```rust
if let Commands::Completions { shell } = &cli.command {
    let mut stdout = std::io::stdout().lock();
    write_shell_completion(*shell, &mut stdout)?;
    return Ok(());
}
```

- `completions` 命令**不加载配置、不初始化日志**，直接输出补全脚本到 stdout
- 这是为了避免日志输出来污染 shell 补全脚本

### 5. 日志初始化

```rust
let subscriber = fmt::Subscriber::builder()
    .with_env_filter(
        EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")),
    )
    .finish();
tracing::subscriber::set_global_default(subscriber).expect("setting default subscriber failed");
```

- 初始化 `tracing` 日志系统
- 优先使用 `RUST_LOG` 环境变量，默认级别为 `info`

### 6. Onboard 向导处理

```rust
if let Commands::Onboard { ... } = &cli.command {
    // 参数验证逻辑...
    let config = if channels_only {
        onboard::run_channels_repair_wizard().await
    } else if interactive {
        onboard::run_wizard(force).await
    } else {
        onboard::run_quick_setup(...).await
    }?;

    // 如果用户选择自动启动渠道
    if std::env::var("ZEROCLAW_AUTOSTART_CHANNELS").as_deref() == Ok("1") {
        channels::start_channels(config).await?;
    }
    return Ok(());
}
```

- Onboard 支持三种模式：
  - `--interactive`: 完整的交互式向导
  - `--channels-only`: 仅修复渠道配置
  - 快速模式: 通过命令行参数直接配置
- 使用 `spawn_blocking` 运行向导，避免 Tokio 运行时冲突
- 验证参数互斥性（如 `--interactive` 和 `--channels-only` 不能同时使用）

### 7. 配置加载与初始化

```rust
let mut config = Config::load_or_init().await?;
config.apply_env_overrides();
observability::runtime_trace::init_from_config(&config.observability, &config.workspace_dir);
```

- 加载或初始化配置
- 应用环境变量覆盖
- 初始化运行时追踪

### 8. OTP 初始化（如果启用）

```rust
if config.security.otp.enabled {
    let config_dir = config.config_path.parent().context("...")?;
    let store = security::SecretStore::new(config_dir, config.secrets.encrypt);
    let (_validator, enrollment_uri) =
        security::OtpValidator::from_config(&config.security.otp, config_dir, &store)?;
    if let Some(uri) = enrollment_uri {
        println!("Initialized OTP secret for ZeroClaw.");
        println!("Enrollment URI: {uri}");
    }
}
```

- 如果启用了 OTP，初始化 OTP 验证器并显示注册 URI

### 9. 命令分发

```rust
match cli.command {
    Commands::Onboard { .. } | Commands::Completions { .. } => unreachable!(),
    Commands::Agent { ... } => agent::run(...).await,
    Commands::Gateway { ... } => gateway::run_gateway(...).await,
    Commands::Daemon { ... } => daemon::run(...).await,
    // ... 其他命令
}
```

- 使用 `match` 语句分发到对应的命令处理函数

---

## CLI 命令概览

| 分类 | 命令 | 用途 |
|:---|:---|:---|
| **初始化与配置** | `onboard` | 初始化工作空间和配置 |
| | `config` | 配置管理 |
| | `completions` | 生成 Shell 补全脚本 |
| **运行时模式** | `agent` | 交互式 AI 对话或单条消息 |
| | `gateway` | 启动网关服务器（Webhooks/WebSockets） |
| | `daemon` | 启动完整守护进程（gateway + channels + 调度器） |
| **系统管理** | `status` | 显示系统状态和配置摘要 |
| | `service` | 管理 OS 服务生命周期（systemd/launchd） |
| | `doctor` | 运行诊断和健康检查 |
| **安全与监控** | `estop` | 紧急停止管理 |
| | `cron` | 定时任务管理 |
| | `auth` | 认证配置文件管理 |
| **AI 配置** | `models` | 模型目录管理 |
| | `providers` | 列出支持的 AI 提供商 |
| **通信渠道** | `channel` | 管理通信渠道（Telegram/Discord/Slack 等） |
| **扩展功能** | `integrations` | 浏览 50+ 集成 |
| | `skills` | 管理用户定义的技能 |
| | `migrate` | 从其他运行时迁移数据 |
| | `memory` | 管理代理内存 |
| | `hardware` | 发现和检查 USB 硬件 |
| | `peripheral` | 管理硬件外设 |

---

## 命令详细说明

### 初始化与配置

#### `onboard` - 初始化配置

启动配置向导，支持快速设置或交互式向导。

```bash
# 快速设置（默认）
zeroclaw onboard --api-key <KEY> --provider <ID> --model <MODEL>

# 交互式向导
zeroclaw onboard --interactive

# 仅修复渠道配置
zeroclaw onboard --channels-only

# 强制覆盖现有配置
zeroclaw onboard --force
```

**参数说明：**
| 参数 | 说明 |
|:---|:---|
| `--interactive` | 运行完整的交互式向导 |
| `--force` | 覆盖现有配置而不确认 |
| `--channels-only` | 仅重新配置渠道（快速修复） |
| `--api-key <KEY>` | API 密钥（快速模式） |
| `--provider <ID>` | 提供商名称（默认：openrouter） |
| `--model <MODEL>` | 模型 ID（快速模式） |
| `--memory <BACKEND>` | 内存后端（sqlite/lucid/markdown/none） |

**安全行为：**
- 如果 `config.toml` 已存在且运行 `--interactive`，向导提供两种模式：
  - 完整配置（覆盖 `config.toml`）
  - 仅更新提供商（保留渠道、隧道、内存等其他设置）
- 在非交互式环境中，现有配置会导致安全拒绝，除非使用 `--force`
- 使用 `--channels-only` 仅轮换渠道令牌/允许列表

#### `config` - 配置管理

```bash
# 导出配置的 JSON Schema
zeroclaw config schema
```

`config schema` 输出完整的 JSON Schema（draft 2020-12）到 stdout，文档化每个可用的配置键、类型和默认值。

#### `completions` - Shell 补全

```bash
zeroclaw completions bash    # Bash
zeroclaw completions fish    # Fish
zeroclaw completions zsh     # Zsh
zeroclaw completions powershell  # PowerShell
zeroclaw completions elvish  # Elvish
```

**特点：**
- 仅输出到 stdout，可直接 source：
  ```bash
  source <(zeroclaw completions bash)
  ```
- 不加载配置、不初始化日志，确保脚本纯净

---

### 运行时模式

#### `agent` - AI 代理模式

启动与 AI 提供商的交互式聊天会话。

```bash
# 交互式会话
zeroclaw agent

# 单条消息模式（不进入交互模式）
zeroclaw agent -m "Summarize today's logs"

# 指定提供商和模型
zeroclaw agent -p anthropic --model claude-sonnet-4-20250514

# 附加硬件外设
zeroclaw agent --peripheral nucleo-f401re:/dev/ttyACM0
```

**参数说明：**
| 参数 | 短选项 | 说明 |
|:---|:---|:---|
| `--message <TEXT>` | `-m` | 单条消息模式 |
| `--provider <ID>` | `-p` | 指定提供商 |
| `--model <MODEL>` | | 指定模型 |
| `--temperature <0.0-2.0>` | `-t` | 采样温度（默认：0.7） |
| `--peripheral <board:path>` | | 附加硬件外设 |

**重要说明：**
- `agent` 是**独立运行模式**，**不需要**先启动 `daemon`
- 与 `daemon` 的区别：
  - `agent`: 前台交互式会话，退出即结束
  - `daemon`: 后台常驻服务，持续运行并接收外部事件
- 在交互式聊天中，可以用自然语言请求路由更改（如"对话使用 kimi，编程使用 gpt-5.3-codex"），助手可以通过 `model_routing_config` 工具持久化此配置

#### `gateway` - 网关服务器

启动 HTTP/WebSocket 网关，接受 Webhook 事件和 WebSocket 连接。

```bash
zeroclaw gateway                  # 使用配置默认值
zeroclaw gateway -p 8080          # 监听 8080 端口
zeroclaw gateway --host 0.0.0.0   # 绑定到所有接口
zeroclaw gateway -p 0             # 随机可用端口
```

**参数说明：**
| 参数 | 短选项 | 说明 |
|:---|:---|:---|
| `--port <PORT>` | `-p` | 监听端口（0 表示随机端口） |
| `--host <HOST>` | | 绑定主机 |

#### `daemon` - 守护进程

启动完整的 ZeroClaw 运行时：网关服务器、所有配置的渠道、心跳监控和 cron 调度器。

```bash
zeroclaw daemon                   # 使用配置默认值
zeroclaw daemon -p 9090           # 网关使用 9090 端口
zeroclaw daemon --host 127.0.0.1  # 仅本地主机
```

**特点：**
- 生产环境或常驻助手的推荐运行方式
- 使用 `zeroclaw service install` 注册为 OS 服务，实现开机自启

---

### 系统管理

#### `status` - 系统状态

显示详细的配置和系统摘要。

```bash
zeroclaw status
```

**输出示例：**
```
🦀 ZeroClaw Status

Version:     0.x.x
Workspace:   /path/to/workspace
Config:      /path/to/config.toml

🤖 Provider:      openrouter
   Model:         (default)
📊 Observability:  stdout
🧾 Trace storage:  write (/path/to/traces)
🛡️  Autonomy:      Limited
⚙️  Runtime:       native
💓 Heartbeat:      every 5min
🧠 Memory:         sqlite (auto-save: on)

Security:
  Workspace only:    true
  Allowed roots:     (none)
  Allowed commands:  ls, cat, echo
  Max actions/hour:  100
  Max cost/day:      $5.00
  OTP enabled:       false
  E-stop enabled:    true

Channels:
  CLI:      ✅ always
  telegram: ✅ configured
  discord:  ❌ not configured

Peripherals:
  Enabled:   yes
  Boards:    2
```

#### `service` - 服务管理

管理用户级 OS 服务生命周期（systemd/launchd）。

```bash
zeroclaw service install      # 安装服务
zeroclaw service start        # 启动服务
zeroclaw service stop         # 停止服务
zeroclaw service restart      # 重启服务
zeroclaw service status       # 查看服务状态
zeroclaw service uninstall    # 卸载服务
```

**参数说明：**
| 参数 | 说明 |
|:---|:---|
| `--service-init <auto/systemd/openrc>` | 指定 init 系统（默认：auto 自动检测） |

#### `doctor` - 诊断工具

运行诊断和新鲜度检查。

```bash
zeroclaw doctor                    # 运行全面诊断
zeroclaw doctor models             # 探测模型目录可用性
zeroclaw doctor models --use-cache # 优先使用缓存目录
zeroclaw doctor traces             # 查询运行时追踪事件
zeroclaw doctor traces --id <ID>   # 查看特定追踪事件
```

**子命令：**
- `models`: 跨提供商探测模型目录并报告可用性
- `traces`: 查询运行时工具/模型诊断（从 `observability.runtime_trace_path` 读取）

---

### 安全与监控

#### `estop` - 紧急停止

管理紧急停止状态和级别。

```bash
# 启动紧急停止（默认 kill-all）
zeroclaw estop

# 指定级别
zeroclaw estop --level network-kill
zeroclaw estop --level domain-block --domain "*.chase.com"
zeroclaw estop --level tool-freeze --tool shell --tool browser

# 查看状态
zeroclaw estop status

# 恢复
zeroclaw estop resume
zeroclaw estop resume --network
zeroclaw estop resume --domain "*.chase.com"
zeroclaw estop resume --tool shell --otp <123456>
```

**级别说明：**
| 级别 | 说明 |
|:---|:---|
| `kill-all` | 完全停止所有操作 |
| `network-kill` | 切断网络访问 |
| `domain-block` | 阻止特定域名访问 |
| `tool-freeze` | 冻结特定工具的使用 |

**注意事项：**
- 需要 `[security.estop].enabled = true`
- 当 `[security.estop].require_otp_to_resume = true` 时，`resume` 需要 OTP 验证
- 如果省略 `--otp`，会自动提示输入

#### `cron` - 定时任务

管理计划任务，支持 cron 表达式、RFC 3339 时间戳、持续时间或固定间隔。

```bash
# 列出所有任务
zeroclaw cron list

# 添加 cron 表达式任务
zeroclaw cron add '0 9 * * 1-5' 'Good morning' --tz America/New_York

# 添加定时任务
zeroclaw cron add-at 2025-01-15T14:00:00Z 'Send reminder'

# 添加间隔任务
zeroclaw cron add-every 60000 'Ping heartbeat'

# 一次性延迟任务
zeroclaw cron once 30m 'Run backup in 30 minutes'

# 管理任务
zeroclaw cron pause <task-id>
zeroclaw cron resume <task-id>
zeroclaw cron remove <task-id>
zeroclaw cron update <task-id> --expression '0 8 * * *'
```

**注意事项：**
- 修改计划的操作需要 `cron.enabled = true`
- 创建计划时的 shell 命令载荷会在任务持久化前通过安全命令策略验证

#### `auth` - 认证管理

管理提供商订阅认证配置文件。

```bash
# OAuth 登录（OpenAI Codex 或 Gemini）
zeroclaw auth login --provider openai-codex
zeroclaw auth login --provider gemini --device-code  # 设备码流程

# 粘贴重定向 URL 完成 OAuth
zeroclaw auth paste-redirect --provider openai-codex

# 粘贴令牌（Anthropic 订阅认证）
zeroclaw auth paste-token --provider anthropic
zeroclaw auth setup-token --provider anthropic       # 交互式

# 刷新令牌
zeroclaw auth refresh --provider openai-codex

# 管理配置文件
zeroclaw auth list          # 列出所有配置文件
zeroclaw auth status        # 显示认证状态
zeroclaw auth use --provider <P> --profile <NAME>  # 设置活动配置文件
zeroclaw auth logout --provider <P> --profile <NAME>  # 移除配置文件
```

---

### AI 配置

#### `models` - 模型管理

管理提供商模型目录。

```bash
zeroclaw models refresh                    # 刷新当前提供商的模型
zeroclaw models refresh --provider <ID>    # 刷新指定提供商
zeroclaw models refresh --all              # 刷新所有支持实时发现的提供商
zeroclaw models refresh --force            # 强制刷新，忽略缓存

zeroclaw models list                       # 列出缓存的模型
zeroclaw models set <MODEL>                # 设置默认模型
zeroclaw models status                     # 显示模型配置和缓存状态
```

**支持的实时刷新提供商：**
`openrouter`, `openai`, `anthropic`, `groq`, `mistral`, `deepseek`, `xai`, `together-ai`, `gemini`, `ollama`, `llamacpp`, `sglang`, `vllm`, `astrai`, `venice`, `fireworks`, `cohere`, `moonshot`, `glm`, `zai`, `qwen`, `nvidia`

#### `providers` - 提供商列表

列出支持的 AI 提供商。

```bash
zeroclaw providers
```

输出包括：
- 提供商 ID（配置中使用）
- 显示名称
- 本地部署标记 `[local]`
- 当前活动标记 `(active)`
- 别名列表

---

### 通信渠道

#### `channel` - 渠道管理

管理通信渠道（Telegram、Discord、Slack、WhatsApp、Matrix、iMessage、Email）。

```bash
zeroclaw channel list              # 列出所有渠道
zeroclaw channel start             # 启动所有配置的渠道
zeroclaw channel doctor            # 运行渠道健康检查
zeroclaw channel bind-telegram <IDENTITY>  # 绑定 Telegram 身份
zeroclaw channel add <type> <json> # 添加渠道
zeroclaw channel remove <name>     # 移除渠道
```

**运行时聊天命令**（Telegram/Discord 渠道服务器运行时可用）：
- `/models` - 列出模型
- `/models <provider>` - 列出指定提供商的模型
- `/model` - 显示当前模型
- `/model <model-id>` - 切换模型
- `/new` - 开始新对话

**热重载：**
渠道运行时会监视 `config.toml` 并热应用以下更新：
- `default_provider`
- `default_model`
- `default_temperature`
- `api_key` / `api_url`（针对默认提供商）
- `reliability.*` 提供商重试设置

---

### 扩展功能

#### `integrations` - 集成浏览

```bash
zeroclaw integrations info <name>   # 查看特定集成详情
```

#### `skills` - 技能管理

管理用户定义的能力（user-defined capabilities）。

```bash
zeroclaw skills list                          # 列出所有技能
zeroclaw skills audit <source_or_name>        # 审计技能安全性
zeroclaw skills install <source>              # 安装技能
zeroclaw skills remove <name>                 # 移除技能
```

**`<source>` 格式：**
- Git 远程：`https://...`, `http://...`, `ssh://...`, `git@host:owner/repo.git`
- 本地文件系统路径

**安全审计：**
`skills install` 在接受技能前始终运行内置的静态安全审计，阻止：
- 技能包内的符号链接
- 脚本文件（`.sh`, `.bash`, `.zsh`, `.ps1`, `.bat`, `.cmd`）
- 高风险命令片段（如 pipe-to-shell）
- 指向技能根目录外的 Markdown 链接、指向远程 Markdown 的链接或指向脚本文件的链接

**技能清单（`SKILL.toml`）：**
支持 `prompts` 和 `[[tools]]`，两者在运行时注入到代理系统提示中，使模型能够遵循技能指令而无需手动阅读技能文件。

#### `migrate` - 数据迁移

从其他代理运行时导入数据。

```bash
zeroclaw migrate openclaw [--source <path>] [--dry-run]
```

#### `memory` - 内存管理

管理代理内存条目。

```bash
zeroclaw memory stats                    # 显示内存统计
zeroclaw memory list                     # 列出内存条目
zeroclaw memory list --category core --limit 10
zeroclaw memory get <key>                # 获取特定条目
zeroclaw memory clear --category conversation --yes  # 清除内存
```

**特点：**
- 支持按类别和会话过滤
- 支持分页
- 批量清除需要确认

#### `hardware` - 硬件发现

发现和检查 USB 硬件。

```bash
zeroclaw hardware discover               # 枚举连接的 USB 设备
zeroclaw hardware introspect /dev/ttyACM0  # 检查特定设备
zeroclaw hardware info --chip STM32F401RETx  # 获取芯片信息
```

**功能：**
- 枚举连接的 USB 设备
- 识别已知的开发板（STM32 Nucleo、Arduino、ESP32）
- 通过 probe-rs / ST-Link 检索芯片信息

#### `peripheral` - 外设管理

管理硬件外设（STM32、RPi GPIO 等）。

```bash
zeroclaw peripheral list                 # 列出所有外设
zeroclaw peripheral add nucleo-f401re /dev/ttyACM0  # 添加外设
zeroclaw peripheral add rpi-gpio native  # 添加 RPi GPIO
zeroclaw peripheral flash --port /dev/cu.usbmodem12345  # 刷写固件
zeroclaw peripheral flash-nucleo         # 刷写 Nucleo 固件
```

**支持的外设：**
- `nucleo-f401re` - STM32 Nucleo 开发板
- `rpi-gpio` - 树莓派 GPIO
- `esp32` - ESP32 开发板
- `arduino-uno` - Arduino Uno

---

## 验证提示

要快速验证文档与当前二进制文件：

```bash
# 查看顶层帮助
zeroclaw --help

# 查看特定命令帮助
zeroclaw <command> --help

# 示例
zeroclaw agent --help
zeroclaw cron --help
```

---

## 环境变量

| 变量 | 说明 |
|:---|:---|
| `ZEROCLAW_CONFIG_DIR` | 配置目录路径 |
| `ZEROCLAW_AUTOSTART_CHANNELS` | 设置为 `1` 在 onboard 后自动启动渠道 |
| `RUST_LOG` | 日志级别（如 `info`, `debug`, `trace`） |
