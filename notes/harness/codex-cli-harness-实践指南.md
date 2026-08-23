# Codex CLI Harness 实践指南

> 分析对象：[openai/codex](https://github.com/openai/codex)（本地 fork：[Willam2004/codex](https://github.com/Willam2004/codex)，经 compare API 验证与上游 `main` 完全一致，无自有改动）
> 版本基线：`rust-v0.149.0`（2026-08-20 发布）；分析日期：2026-08-23
> 仓库规模：⭐ 114k+，主语言 Rust，Apache-2.0
> 关联笔记：[hanlee-agent-not-model.md](../papers/hanlee-agent-not-model.md) —— 该论文提出 `Agent = Model + Harness + Tools + Environment + Rewards + Evaluation`，Codex 正是其中 **Harness（模型外壳/执行框架）** 概念的工业级实现样本

---

## 一、Codex 是什么，Harness 指什么

Codex CLI 是 OpenAI 官方的终端编程智能体（"Lightweight coding agent that runs in your terminal"）。用 Han Lee 的框架拆解：

| Han Lee 框架组件 | 在 Codex 中的对应 |
|---|---|
| Model（模型） | GPT 系列（经 Responses API），可换模型/推理强度 |
| **Harness（外壳）** | **codex-rs core：代理循环、上下文管理、审批与沙箱、会话持久化** |
| Tools（工具） | shell 执行、apply_patch、web_search、MCP 工具、子代理 |
| Environment（环境） | 本地文件系统 + git 仓库（受沙箱约束） |
| Rewards / Evaluation（评估） | Guardian 自动审查、`codex review`、CI 集成 |

Harness 的职责可以概括为四件事：

1. **驱动循环**：把用户输入变成一轮轮「模型请求 → 工具调用 → 结果回填」的 Turn 循环，直到任务完成；
2. **执行工具**：真正跑命令、打补丁的是 harness，不是模型——模型只产生调用意图；
3. **守住边界**：沙箱（内核级隔离）+ 审批（谁来批准）+ 执行策略（静态规则）三层闸门；
4. **管理状态**：上下文压缩、会话录制（rollout）、恢复/分叉、跨会话记忆。

## 二、整体架构

### 2.1 分层视图

```
┌────────────────────────── 前端（UI 层）──────────────────────────┐
│  codex (TUI 交互)   codex exec (无头)   IDE/桌面 (app-server)    │
│  外部 MCP 客户端 ← codex mcp-server     Codex Cloud (codex cloud) │
└──────────────────────────────┬───────────────────────────────────┘
                               │ Op / Event 协议（SQ 提交队列 / EQ 事件队列）
┌──────────────────────────────▼───────────────────────────────────┐
│                    Codex 核心引擎（codex-rs/core）                │
│  Session（会话配置）→ Task（一次任务）→ Turn（单轮迭代）           │
│  工具编排：exec_command / apply_patch / update_plan / MCP / 子代理 │
│  安全层：execpolicy → approval policy (+Guardian) → sandbox       │
│  上下文管理：自动压缩 / 记忆注入 / AGENTS.md 拼装                  │
└──────────────────────────────┬───────────────────────────────────┘
                               │ Responses API（SSE 流式）
                        ┌──────▼──────┐
                        │  Model (LLM) │
                        └─────────────┘
```

### 2.2 核心实体模型（源自 `codex-rs/docs/protocol_v1.md`）

- **Codex**：核心引擎，本地运行，通过 SQ/EQ 消息队列对与任意 UI 通信；
- **Session**：当前配置与状态，由 `ConfigureSession` 初始化，可随时重配（重配会中止运行中的任务）；
- **Task**：一次用户输入触发的执行，由多个 Turn 组成；同一时刻最多一个 Task；
- **Turn**：一次「请求模型 → 流式收集 → 执行工具/打补丁 → 必要时暂停等审批」的迭代，上一轮输出是下一轮输入；无输出的 Turn 结束 Task。

每个 Turn 完成会带回 `response_id`（对应 OpenAI `/responses` 端点），可存档用于**恢复（resume）**或在任意历史点**分叉（fork）**——这是 Codex 会话能力的根基。

### 2.3 代码地图（codex-rs/ 工作区，100+ crate）

| 模块 | 职责 |
|---|---|
| `core` | 代理循环、工具实现、Guardian、压缩、配置运行时 |
| `tui` | 交互式终端界面（ratatui） |
| `exec` | 无头模式 `codex exec` 及 `codex review` |
| `app-server` / `app-server-protocol` | IDE/桌面集成协议（JSON-RPC，v1/v2） |
| `mcp-server` | 把 Codex 自身暴露为 MCP server |
| `protocol` | Op/Event、沙箱/审批等全部线上类型 |
| `sandboxing` / `linux-sandbox` / `bwrap` / `windows-sandbox-rs` | 各平台沙箱后端 |
| `execpolicy` | Starlark 命令静态规则引擎 |
| `skills` / `hooks` / `plugin` / `core-plugins` | 扩展体系 |
| `memories` | 跨会话长期记忆 |
| `rollout` / `thread-store` / `history` | 会话持久化与恢复 |
| `apply-patch` | 自定义补丁格式（模型写补丁、harness 应用） |
| `codex-home` / `config` | `~/.codex` 目录与分层配置加载 |

## 三、安装与认证

### 3.1 安装

```bash
# 官方一键脚本（macOS / Linux；默认从 releases.openai.com 下载，失败回退 GitHub Releases）
curl -fsSL https://chatgpt.com/codex/install.sh | sh

# 或包管理器
npm install -g @openai/codex
brew install --cask codex
```

Windows 用 `irm https://chatgpt.com/codex/install.ps1 | iex`；也可从 [GitHub Releases](https://github.com/openai/codex/releases/latest) 下载平台二进制（如 `codex-aarch64-apple-darwin.tar.gz`）。

日常维护命令：`codex update`（自更新）、`codex doctor`（诊断安装/配置/认证/运行时健康）、`codex completion <shell>`（补全脚本）。

### 3.2 认证

- 推荐：运行 `codex` 后选 **Sign in with ChatGPT**（Plus/Pro/Business/Edu/Enterprise 计划内使用）；
- 也可用 API key（`OPENAI_API_KEY`）；
- 凭据存储由 `cli_auth_credentials_store` 控制：`file`（默认，`~/.codex/auth.json`）/ `keyring`（系统钥匙串）/ `auto` / `ephemeral`（仅内存）；
- `codex login` / `codex logout` 管理登录态。

## 四、配置系统

### 4.1 `~/.codex` 目录布局

```
~/.codex/                    # 可被 CODEX_HOME 环境变量覆盖
├── config.toml              # 主配置
├── hooks.toml               # 可选：独立 hooks 文件
├── AGENTS.md                # 全局个人指令（AGENTS.override.md 优先）
├── auth.json                # 登录凭据（file 模式）
├── history.jsonl            # 输入历史（[history] 可关）
├── log/                     # 日志（log_dir）
├── sessions/                # 会话 rollout（JSONL）
├── archived_sessions/       # 归档会话
├── skills/                  # 用户技能；.system/ 为内置技能
├── plugins/                 # 已安装插件
└── <state db>               # SQLite 状态库（CODEX_SQLITE_HOME 可移走）
```

### 4.2 配置分层（低 → 高优先级）

内置默认 → 云端/组织托管配置（requirements/managed，含 MDM）→ 用户 `~/.codex/config.toml` → 项目级 `.codex/config.toml`（受项目信任约束，未受信项目的项目指令会被忽略）→ CLI/会话级覆盖。TUI 中 `/debug-config` 可查看每个 key 由哪一层决定。

### 4.3 Profiles：一套配置多个场景

```toml
profile = "work"                    # 当前激活的 profile；CLI 用 -p/--profile 切换

[profiles.work]
model = "gpt-5"
approval_policy = "on-request"
sandbox_mode = "workspace-write"
model_reasoning_effort = "high"

[profiles.oss]
sandbox_mode = "read-only"
```

Profile 支持 `model`、`model_provider`、`approval_policy`、`sandbox_mode`、`model_reasoning_effort`、`personality`、`tools`、`web_search`、`features`、`tui` 等字段子集。另有新的 profile-v2 机制：`~/.codex/<name>.config.toml` 整文件级 profile。

### 4.4 常用配置项速查（`config.toml` 顶层共约 95 个 key）

| Key | 说明 |
|---|---|
| `model` / `model_provider` / `model_providers` | 模型与提供方；可配自定义 OpenAI 兼容端点 |
| `model_reasoning_effort` / `model_reasoning_summary` / `model_verbosity` | 推理强度 / 摘要展示 / 输出详略 |
| `model_context_window` / `model_auto_compact_token_limit` | 上下文窗口 / 自动压缩阈值 |
| `approval_policy` / `approvals_reviewer` / `sandbox_mode` | 安全三件套（见第六节） |
| `review_model` | Guardian/审查专用模型 |
| `instructions` / `developer_instructions` | 内联指令 |
| `project_doc_max_bytes`（默认 32768）/ `project_doc_fallback_filenames` / `project_root_markers`（默认 `[".git"]`） | AGENTS.md 发现行为 |
| `history.persistence = "save-all" \| "disable"` | 输入历史 |
| `notify` | 回合结束通知钩子 |
| `web_search`（`disabled/cached/indexed/live`） | 联网搜索 |
| `personality` | 沟通风格 |
| `oss_provider` / `--oss` / `--local-provider` | 本地模型（Ollama / LM Studio 等） |
| `tui.*` | TUI 外观与行为 |

完整参考：仓库内 `codex-rs/core/config.schema.json`（JSON Schema），官方文档 <https://developers.openai.com/codex/config-reference>。

### 4.5 AGENTS.md：给代理的项目说明书

加载顺序（拼接注入提示词）：

1. **全局**：`~/.codex/AGENTS.override.md` 优先，否则 `~/.codex/AGENTS.md`（取第一个非空，不拼接）；
2. **项目**：从项目根（由 `project_root_markers` 识别，默认找 `.git`）向下到当前目录，逐层拼接各级 `AGENTS.md`；同目录存在 `AGENTS.override.md` 时覆盖该层（适合放本地未提交的偏好）；
3. 无 `AGENTS.md` 时尝试 `project_doc_fallback_filenames`（如 `CLAUDE.md`、`README.md`）。

总量受 `project_doc_max_bytes`（默认 32 KiB）限制。TUI 中 `/init` 可让 Codex 为当前仓库自动生成一份 AGENTS.md。

**实践建议**：AGENTS.md 写「构建/测试命令、代码规范、仓库雷区」这类模型猜不到的硬事实，别写废话——它每次会话都要花 token。

## 五、代理循环与工具系统

### 5.1 内置工具

| 工具 | 作用 |
|---|---|
| `exec_command`（shell）/ `write_stdin` | 执行命令、向运行中的进程写输入（主力工具） |
| `apply_patch` | 应用 Codex 自定义补丁格式做文件增删改 |
| `update_plan` | 生成/更新任务计划（TUI 中可视化） |
| `view_image` | 读取图片（截图、图表） |
| `request_user_input` | 向用户提问（可带选项） |
| `request_permissions` | 请求提升权限 |
| `web_search` | 托管联网搜索（可配 `allowed_domains`） |
| `list_mcp_resources` / `read_mcp_resource` 等 | MCP 资源访问 |
| `spawn_agent` / `send_message` / `wait_agent` … | 子代理协作（见第九节） |
| `current_time` / `sleep` / `new_context_window` / `get_context_remaining` | 时间与上下文预算辅助 |

工具输出受 `tool_output_token_limit` 截断保护；`tool_result_log.max_bytes` 控制日志留痕。

### 5.2 上下文管理

- 达到 `model_auto_compact_token_limit` 自动压缩（`model_auto_compact_token_limit_scope = "full" | "post-prefix"`），压缩提示词可用 `compact_prompt` 定制；TUI 中 `/compact` 手动触发；
- `/status` 查看当前 token 用量；`/export` 导出会话为 Markdown；
- 实验性：`new_context_window` 工具支持「预算制会话」主动开新上下文窗口。

### 5.3 TUI 常用斜杠命令（节选）

```
/init 生成 AGENTS.md        /plan 切换计划模式        /model 选模型与推理强度
/compact 压缩上下文          /review 审查当前改动       /diff 查看 git diff（含未跟踪）
/resume /fork /archive      /agents 查看活跃代理会话    /skills /hooks /mcp /plugins
/permissions 调权限          /approvals 相关审批设置     /usage 账户用量
/status 会话与 token 状态    /export 导出 Markdown       /import 从 Claude Code 迁移
/cd /pwd 切换目录            /vim Vim 模式             /theme /statusline /pets 个性化
```

其中 `/import` 可以一键导入 Claude Code 的设置、当前项目与最近会话——从 Claude Code 迁移过来的实用入口。

## 六、安全模型：三层闸门

这是 Codex harness 最值得学习的部分。一条命令从模型产生到真正执行，要过三关：

```
模型想执行 `rm -rf build/`
   │
   ▼ ① execpolicy（静态规则，Starlark）
   Allow → 直接放行 │ Prompt → 进入审批 │ Forbidden → 硬拒绝
   ▼ ② approval_policy（谁来决定）+ approvals_reviewer（人 or Guardian）
   批准 / 拒绝（可附「以后都允许」→ 写回 execpolicy 修正案）
   ▼ ③ sandbox（内核级隔离，即使批准了也限制爆炸半径）
   Seatbelt / Landlock+seccomp / Restricted Token
```

### 6.1 沙箱（`sandbox_mode`）

| 模式 | 含义 |
|---|---|
| `read-only`（默认） | 只能读，不能写文件、不能改状态 |
| `workspace-write` | 可写当前工作区（+ `--add-dir` 附加目录）；网络默认禁 |
| `danger-full-access` | 完全不设限（危险） |

`[sandbox_workspace_write]` 细粒度选项：`writable_roots`（额外可写路径）、`network_access = true`（放行网络）、`exclude_tmpdir_env_var` / `exclude_slash_tmp`（排除 tmp 目录）。

平台后端（自动选择）：

- **macOS**：Seatbelt（`sandbox-exec`，仓库内有多份 `.sbpl` 策略文件）；
- **Linux**：Landlock + seccomp（`codex-linux-sandbox` 辅助进程，另内置 bubblewrap 后端）；
- **Windows**：受限令牌（`windows_sandbox_level = Disabled(默认) | RestrictedToken | Elevated`）。

### 6.2 审批（`approval_policy`）

| 策略 | 行为 |
|---|---|
| `untrusted` | 除 execpolicy 显式允许外，每条命令都要批 |
| `on-request`（默认，别名 `on-failure`） | 模型自行判断何时请求审批 |
| `granular` | 按类别细控（沙箱审批/规则/技能审批/权限请求/MCP elicitation 各自开关） |
| `never` | 永不弹窗，失败直接回给模型 |

内置预设：`read-only`（on-request + 只读）、`auto`（默认，on-request + workspace-write）、`full-access`（never + 不设限）。对应 CLI 快捷开关：

- `--approve-for-me`（别名 `--not-so-yolo`）= `approvals_reviewer=auto_review` + `on-request` + `workspace-write`，即「让 Guardian 代我审批」；
- `--dangerously-bypass-approvals-and-sandbox`（别名 `--yolo`）= 全绕过，**仅限外部已隔离的环境**（容器/虚拟机）。

### 6.3 Guardian：用子代理代审

`approvals_reviewer = "auto_review"`（旧名 `guardian_subagent`）时，审批请求不再弹给人，而是交给一个专职 **Guardian 子代理**：

- 它继承父会话配置、重建精简上下文，用独立审查模型（可用 `review_model` 单指）输出严格 JSON 评估：`risk_level`（low/medium/high/critical）× `user_authorization`（推断用户授权程度）→ `outcome: allow/deny` + `rationale`；
- 安全设计：**fail-close**（超时 90s 或输出不合规即拒）、单轮最多 3 次重试、连续被拒触发熔断回退，防止代理被诱导后无限闯关；
- Guardian 自己的会话只保留 `exec_command` / `write_stdin` / `view_image` 最小工具集；
- TUI `/auto-review` 可对最近一次自动拒绝放行重试。

**实践建议**：个人机器日常用默认 `on-request` + 人工审批；批量自动化（CI、脚本）在容器里用 `--approve-for-me` 或 `--yolo`；重要仓库可常开 Guardian 当「第二双眼睛」。

### 6.4 execpolicy：命令白名单引擎

`codex-rs/execpolicy` 是一个 Starlark 规则引擎，在审批之前做静态裁决（Allow / Prompt / Forbidden）。规则形如 `prefix_rule(pattern=["git", "status"], decision="allow")`，支持 `network_rule`（按域名放行网络）。审批弹窗里选「总是允许」会把修正案追加进策略文件。可用隐藏命令自检：`codex execpolicy check --rules <file> <cmd>`。

### 6.5 permissions：声明式权限档案

`[permissions.<id>]` 可定义命名权限档案：文件系统按 glob 给 `read`/`write`/`deny`，网络可配开关与代理；支持 `extends` 继承。内置档案 `:read-only`、`:workspace`、`:danger-full-access`。TUI `/permissions` 可视化切换。

## 七、会话管理

- 每个会话以 **JSONL rollout** 持久化在 `~/.codex/sessions/`（归档在 `archived_sessions/`），首行是 `SessionMeta`，其后逐条记录消息/工具调用/压缩点；`/rollout` 打印当前会话文件路径；
- `codex resume`（`--last` 续最近一次）/ `codex fork`：基于历史恢复或分叉；恢复时可选 `current`（用启动目录）或 `session`（用录制时的 cwd）；
- `codex queue`：给一个正在跑的会话追加消息；`codex archive` / `unarchive` / `delete` 管理会话生命周期；`codex agents` 浏览本地 app-server 守护进程上的所有代理会话；
- **记忆（`[memories]`）**：后台进程在启动时回顾近期空闲会话（`max_rollout_age_days=10`、`min_rollout_idle_hours=6`、每次最多 `max_rollouts_per_startup=2` 个 rollout），抽取并整合长期记忆，注入后续会话的提示词；`use_memories` / `generate_memories` 开关，`/memories` 管理，存储于本地 SQLite。

## 八、扩展体系

### 8.1 MCP（双向）

**接外部 MCP server**（给 Codex 加工具）：

```toml
# ~/.codex/config.toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_TOKEN = "..." }
```

或 `codex mcp add <name> -- <command> <args...>`；`codex mcp list|get|remove|login|logout` 管理（支持 OAuth 登录流）。`/mcp` 查看连接状态与工具。

**把 Codex 暴露为 MCP server**（`codex mcp-server`，实验性）：外部 MCP 客户端可通过 `thread/start`、`thread/resume`、`thread/fork`、`turn/start`、`turn/steer`、`turn/interrupt`、`model/list`、`config/read` 等 RPC 驱动完整的 Codex 引擎——这是把 Codex 嵌进自有工作流的官方通道。调试：`npx @modelcontextprotocol/inspector codex mcp-server`。

### 8.2 Skills（技能）

技能 = 一份带 YAML frontmatter 的 `SKILL.md`（`name` + `description` + 正文指令），按需注入提示词：

```yaml
---
name: my-skill
description: One-line description shown to the model.
metadata:
  short-description: Even shorter.
---
（正文：给模型的操作指南/流程/脚本说明）
```

- 存放位置：`~/.codex/skills/`（用户）、项目内（插件/项目技能）、`~/.codex/skills/.system/`（内置，随版本更新：`skill-creator`、`skill-installer`、`plugin-creator`、`review-agent`、`openai-docs`、`imagegen`）；
- 触发：显式 `@skill-name` 或 `/skill-name`；也支持隐式触发（命令匹配技能声明的访问模式，`policy.allow_implicit_invocation` 控制）；
- 配置：`[skills]` 下 `include_instructions`、`max_context_tokens`（目录注入预算）、`[skills.bundled] enabled`、`[[skills.config]]` 按名启用/停用；`/skills` 管理；
- 用内置 `skill-creator` 技能可以让 Codex 帮你写新技能（设计哲学：只写模型猜不到的、会改变决策的信息，别写正确的废话）。

### 8.3 Hooks（生命周期钩子）

11 个事件：`PreToolUse`、`PermissionRequest`、`PostToolUse`、`PreCompact`、`PostCompact`、`SessionStart`、`SessionEnd`、`UserPromptSubmit`、`SubagentStart`、`SubagentStop`、`Stop`。

```toml
# config.toml 内（或独立 hooks.toml）
[[hooks.PreToolUse]]
matcher = "shell"                    # 可选：按工具名过滤

[[hooks.PreToolUse.hooks]]
type = "command"                     # command | mcp_tool | prompt | agent
command = "scripts/lint-check.sh"
timeout = 30
```

钩子结果 `Success / FailedContinue / FailedAbort`（最后者可中止触发操作）。安全机制：钩子首次运行需建立信任（记录 `trusted_hash`，内容变更会重新提示），自动化场景可用 `--dangerously-bypass-hook-trust` 跳过；`/hooks` 可视化管理。管理员可用 `requirements.toml` 的 `allow_managed_hooks_only = true` 只允许受管钩子。

### 8.4 Plugins（插件）与 Marketplace

插件 = 打包的扩展集合，可同时携带 **skills + MCP servers + hooks + app connectors**（manifest 声明 `paths.*`）：

```bash
codex plugin add <plugin>[@<marketplace>]
codex plugin list [--available] [--json]
codex plugin remove <plugin>
codex plugin marketplace add|list|remove|upgrade
```

配置面 `[plugins.<id>]`：启停插件、逐 MCP server 设置 `default_tools_approval_mode` 与 `enabled_tools`/`disabled_tools` 白/黑名单。`/plugins` 浏览。

## 九、协作模式与多代理

### 9.1 协作模式（Collaboration Modes）

- **default**：常规结对编程模式；
- **plan**（`/plan` 切换）：三阶段对话式规划——先讨论、再定稿、确认前只允许非变更操作；适合大改动前先对齐方案。可用 `plan_mode_reasoning_effort` 单独设置计划阶段推理强度。

### 9.2 子代理（Subagents）

Codex 核心内建多代理编排（`collaboration` 工具命名空间）：`spawn_agent`、`send_message`、`followup_task`、`wait_agent`、`interrupt_agent`、`list_agents`、`close_agent`、`resume_agent`。

- 用 `[agents.roles.<name>]` 定义角色（`description` + 独立 `config_file`），`agents_enabled`、`agent_max_threads` 控制开关与并发；
- 触发策略 `ExplicitRequestOnly`（默认，仅按用户要求派生）/ `Proactive`（主动拆分）/ 自定义；
- TUI `/agents` 看全部活跃会话、`/multi-agents` 在当前会话的子代理间切换；子代理生命周期有 `SubagentStart`/`SubagentStop` 钩子和独立的 Guardian 审查线程。

**实践建议**：并行独立子任务（多模块修改、批量调研）才值得拆子代理；每个子代理都是独立上下文，拆得太碎反而丢失全局信息。

## 十、无头模式与自动化

### 10.1 `codex exec`（别名 `e`）

```bash
# 基本：非交互执行，完成后输出结果
codex exec "给这个函数补单元测试并跑通"

# 常用参数
-m/--model <model>              # 指定模型
-p/--profile <name>             # 指定 profile
-s/--sandbox <mode>             # read-only | workspace-write | danger-full-access
--approve-for-me                # Guardian 代审（= --not-so-yolo）
--yolo                          # 绕过一切审批与沙箱（仅限隔离环境）
--json                          # stdout 输出 JSONL 事件流（机器可读）
-o/--output-last-message FILE   # 把最终回复写入文件
--output-schema FILE            # 约束最终输出符合 JSON Schema
--skip-git-repo-check           # 允许在非 git 目录运行
--ephemeral                     # 不落盘会话记录
--ignore-user-config            # 跳过 ~/.codex/config.toml（CI 纯净环境）
--ignore-rules                  # 跳过 .rules 执行策略
--cd <dir> / --add-dir <dir>    # 指定工作目录 / 附加可写目录
-i/--image <file>               # 附图输入
```

子命令：`codex exec resume` / `codex exec fork`（无头恢复/分叉历史会话）。

### 10.2 `codex review`：代码审查

```bash
codex review --uncommitted           # 审查暂存+未暂存+未跟踪改动
codex review --base main             # 相对某分支的 diff
codex review --commit <sha> --title "..."
```

TUI 内 `/review` 同义。配合 `review_model` 可指定审查专用模型。

### 10.3 云任务与远程控制

- `codex cloud`（实验性）：对接 Codex Cloud 任务——`exec`（`--env`/`--branch`/`--attempts 1-4` best-of-N）、`status`、`list`、`diff`、`apply`（把云端任务产生的改动拉回本地）；
- `codex exec-server`（实验性）：独立的常驻执行服务（JSON-RPC/WebSocket 管理环境、文件系统与进程）；
- `codex app-server` + `codex remote-control`：IDE/桌面协议端与带远程控制的守护进程。

### 10.4 其他实用命令

- `codex apply`（别名 `a`）：把最近一次代理产生的 diff 以 `git apply` 方式落到当前工作树（适合在别处跑完代理、回本地落改动）；
- `codex sandbox -- <cmd>`：在 Codex 沙箱里跑任意命令（把沙箱当工具用）。

### 10.5 CI 集成范式

```yaml
# GitHub Actions 示例：让 Codex 在无头模式修复失败用例（跑在一次性容器里）
- name: Codex auto-fix
  run: |
    codex exec --yolo --skip-git-repo-check \
      --output-last-message result.md \
      "CI 测试失败，请定位并修复，然后重跑测试确认"
```

要点：容器/一次性 VM 内才考虑 `--yolo`；需要交互审批的托管环境用 `--approve-for-me`；`--json` + `-o` 便于把结果喂给下游步骤。

## 十一、实践工作流清单

**新项目接入（5 分钟）**

1. `codex` 进入项目，跑一次 `/init` 生成 AGENTS.md，人工校对构建/测试命令与雷区；
2. 决定安全档位：日常 `workspace-write` + `on-request`；只读分析用默认 `read-only`；
3. 有外部依赖就 `codex mcp add`（如 GitHub、数据库）；
4. 团队共性流程沉淀成 skill 放 `~/.codex/skills/` 或项目插件。

**日常循环**

1. 大改动先 `/plan` 对齐方案，确认后切 default 执行；
2. 信任仓库可用 `--approve-for-me` 减少打断，保留 Guardian 兜底；
3. 平行子任务让主代理 `spawn_agent` 拆分，`/agents` 总览；
4. 上下文变慢就 `/compact`；任务跨度大就依赖 `[memories]` 跨会话记忆;
5. 提交前 `codex review --uncommitted` 自审一遍。

**安全红线**

- 宿主机上永远不要 `--yolo`；它只属于容器/虚拟机/一次性环境；
- 谨慎授予网络权限（`network_access`、execpolicy `network_rule` 按域名最小放行）；
- 钩子与插件来自社区时要审查来源（信任哈希机制会提示变更）；
- 敏感目录用 `read-only` 预设或 `permissions` 档案显式 `deny`。

**排障**

- `codex doctor` 首诊；`~/.codex/log/` 看日志；`/debug-config` 查配置分层；`/status` 看 token；
- 会话出问题可以 `codex resume` 回历史任意点 `fork` 重来。

## 十二、延伸阅读与来源索引

| 主题 | 来源 |
|---|---|
| 官方文档 | <https://developers.openai.com/codex>（config / security / skills / noninteractive 等） |
| 协议规范（Session/Task/Turn、SQ/EQ） | 仓库 `codex-rs/docs/protocol_v1.md` |
| MCP server 接口 | 仓库 `codex-rs/docs/codex_mcp_interface.md` |
| 配置全集 | 仓库 `codex-rs/core/config.schema.json` |
| 沙箱后端 | `codex-rs/sandboxing/`、`codex-rs/linux-sandbox/`、`codex-rs/windows-sandbox-rs/` |
| Guardian 审查 | `codex-rs/core/src/guardian/` |
| 工具实现 | `codex-rs/core/src/tools/`（`spec_plan.rs` 为工具装配入口） |
| CLI 定义 | `codex-rs/cli/src/main.rs`、`codex-rs/utils/cli/src/shared_options.rs` |
| 上游仓库 | <https://github.com/openai/codex>（本笔记 fork：<https://github.com/Willam2004/codex>） |

> 本笔记基于 2026-08-23 的 openai/codex `main` 分支源码分析；Codex 迭代很快（发布节奏约每 2-3 天一个 0.149 → 0.15x 的版本），具体行为以运行时 `codex doctor` 与官方文档为准。
