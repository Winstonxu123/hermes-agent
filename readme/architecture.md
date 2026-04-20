# 系统架构

本文档深入讲解 Hermes Agent 的内部架构，帮助你理解代码是如何组织的、各模块之间如何协作。

---

## 这份文档适合什么时候读

如果你是第一次进入仓库，建议先读：

1. [simple.md](./simple.md)
2. [development-guide.md](./development-guide.md)

再来看这篇。因为这篇默认你已经知道：

- Hermes 有 CLI 和 gateway 两个主要入口
- Hermes 的核心是 `AIAgent` 工具调用循环
- Tools、Skills、Plugins 不是一回事

如果你今天只想做一个小改动，这篇不用一次读完。优先看：

- “核心运行循环”
- “模块依赖关系”
- “工具系统设计”

---

## 新人应该先抓住的 3 条主线

### 1. `run_agent.py` 是主编排器

模型、消息、工具、压缩、记忆、会话持久化，最后都汇总到这里。

### 2. `model_tools.py` + `tools/registry.py` 是工具入口

“模型为什么能看到某个工具”以及“这个工具为什么没暴露出来”的问题，大多要回到这里。

### 3. `cli.py` / `gateway/run.py` 是两种运行模式

同一个 agent 内核分别被终端和消息平台驱动。很多问题其实是入口差异，而不是 agent 核心坏了。

---

## 一条请求怎么走完整条链路

```text
CLI 或 Gateway 收到输入
-> 创建 / 恢复 AIAgent
-> 组装 system prompt 和 tool definitions
-> 调 LLM
-> 如果返回 tool_calls，就分发执行工具
-> 把工具结果追加到消息历史
-> 再调 LLM
-> 返回最终回复
-> 持久化 session / 记忆 / 轨迹
```

调试时可以先判断问题卡在哪一段：

- 输入进入前：CLI / gateway
- API 请求前：prompt / tools / config
- tool call 中：registry / handler / env
- 返回后：memory / compression / persistence / display

---

## 核心运行循环

Hermes Agent 的心脏是一个**工具调用循环**（Tool-Calling Loop）。整个流程如下：

```
用户输入消息
    │
    ▼
AIAgent._run_agent_loop()
    │
    ├── 1. 构建系统提示词（prompt_builder.py）
    │      ├── 智能体身份（DEFAULT_AGENT_IDENTITY）
    │      ├── 平台提示（OS 相关指导）
    │      ├── 技能索引（可用 Skills 列表）
    │      ├── 上下文文件（.hermes.md、SOUL.md）
    │      └── 记忆注入（持久化记忆内容）
    │
    ├── 2. 构建 API 请求参数
    │      ├── 模型名称 & 提供商
    │      ├── 消息历史
    │      ├── 工具定义列表（JSON Schema）
    │      └── 推理配置（reasoning effort）
    │
    ├── 3. 调用 LLM（OpenAI 兼容 API）
    │
    ├── 4. 解析响应
    │      │
    │      ├── [包含 tool_calls] ──► 执行工具
    │      │      ├── 通过 registry 分发到具体 handler
    │      │      ├── 将工具结果加入消息历史
    │      │      └── 回到步骤 3（继续循环）
    │      │
    │      └── [纯文本响应] ──► 返回最终回答
    │             ├── 持久化会话到 SQLite
    │             └── 记录轨迹（可选，用于 RL 训练）
    │
    └── 5. 上下文压缩（如接近 token 上限）
           └── 智能摘要中间对话，保留首尾
```

**关键文件：** `run_agent.py` — `AIAgent` 类（大型核心编排文件）

---

## 模块依赖关系

```
┌─────────────────────────────────────────────────────┐
│              run_agent.py (AIAgent)                  │
│        核心编排：对话循环、工具分发、会话管理           │
└────────┬──────────────────────────────┬─────────────┘
         │                              │
   ┌─────▼──────────┐          ┌───────▼──────────┐
   │  agent/         │          │  hermes_cli/      │
   │  内部模块       │          │  CLI 命令          │
   │  ├ prompt_      │          │  ├ main.py        │
   │  │ builder.py   │          │  ├ setup.py       │
   │  ├ context_     │          │  ├ commands.py    │
   │  │ compressor   │          │  └ ...            │
   │  ├ memory_      │          └───────┬──────────┘
   │  │ manager.py   │                  │
   │  ├ model_       │          ┌───────▼──────────┐
   │  │ metadata.py  │          │  cli.py           │
   │  └ ...          │          │  交互式 TUI       │
   └─────┬──────────┘          └──────────────────┘
         │
   ┌─────▼──────────────────────────────────────┐
   │  model_tools.py                             │
   │  工具发现 & 分发编排                         │
   │  ├ _discover_tools()  导入所有工具模块       │
   │  ├ get_tool_definitions()  获取 Schema      │
   │  └ handle_function_call()  路由执行          │
   └─────┬──────────────────────────────────────┘
         │
   ┌─────▼──────────┐     ┌───────────────────┐
   │  tools/         │     │  gateway/          │
   │  registry.py    │     │  消息平台网关       │
   │  (工具注册中心)  │     │  ├ run.py          │
   │                 │     │  ├ session.py      │
   │  50+ 工具实现    │     │  ├ platforms/      │
   │  ├ web_tools    │     │  │  ├ telegram.py  │
   │  ├ terminal     │     │  │  ├ discord.py   │
   │  ├ file_tools   │     │  │  ├ slack.py     │
   │  ├ browser      │     │  │  └ ...          │
   │  ├ vision       │     │  └ delivery.py     │
   │  └ ...          │     └───────────────────┘
   └────────────────┘
```

---

## 核心模块详解

### 1. `run_agent.py` — 智能体核心

这是整个项目最核心的文件，包含 `AIAgent` 类。

**关键职责：**
- 管理多轮对话的消息历史
- 解析 LLM 返回的工具调用请求，分发执行
- 支持并行工具执行（分析安全性后决定是否并行）
- 当 token 接近上限时触发上下文压缩
- 会话持久化到 SQLite 数据库
- 轨迹记录（用于 RL 训练数据生成）

**关键类：**
- `AIAgent` — 主编排器
- `IterationBudget` — 线程安全的迭代计数器，防止工具调用无限循环

**你需要了解的：** 如果你想修改智能体的核心行为（比如改变工具调用策略、调整压缩逻辑），这是你需要修改的文件。

### 2. `agent/prompt_builder.py` — 提示词构建

负责组装发送给 LLM 的系统提示词。采用**无状态函数**设计。

**组装的内容：**
1. 智能体身份（"You are Hermes Agent..."）
2. 平台提示（根据 OS 给出终端操作指导）
3. 技能索引（列出所有可用 Skill 的名称和描述）
4. 上下文文件（`.hermes.md`、`SOUL.md`、`AGENTS.md`）
5. 记忆块（从 Memory Provider 获取的持久化记忆）
6. 模型特定指导（如 GPT 专用的执行纪律提示）

**安全特性：** 内置提示词注入检测，扫描上下文文件中的恶意指令模式。

### 3. `agent/context_compressor.py` — 上下文压缩

当对话过长、接近模型的 token 上限时，自动压缩历史消息。

**压缩算法：**
1. **预处理** — 将旧的工具结果替换为占位符（无需 LLM，低成本）
2. **边界保护** — 保留对话开头（系统提示 + 首轮交互）
3. **尾部保护** — 保留最近约 20K token 的内容（最新上下文最有价值）
4. **结构化摘要** — 用 LLM 将中间对话摘要为：目标、约束、进度、关键决策、下一步
5. **增量更新** — 再次压缩时更新已有摘要，而非重新生成

### 4. `agent/model_metadata.py` — 模型元数据

解析各种 LLM 的上下文窗口大小、定价等信息。

**数据来源（按优先级）：**
1. models.dev API（覆盖 200+ 模型家族）
2. OpenRouter API
3. 自定义端点的模型列表
4. 内置默认值字典
5. 上下文探测（从 128K 向下逐级尝试）
6. 兜底默认 128K

### 5. `agent/memory_manager.py` — 记忆管理

管理内置 + 外部记忆提供商，支持跨会话持久化。

**工作流程：**
- 对话开始前：`prefetch_all()` 预取相关记忆
- 对话结束后：`sync_all()` 持久化新记忆
- 记忆内容用 `<memory-context>` 标签包裹注入提示词

---

## 工具系统设计

### 自注册模式

每个工具文件在被 import 时自动向 `tools/registry.py` 注册自己：

```python
# tools/web_tools.py 中的注册示例
from tools.registry import registry

registry.register(
    name="web_search",
    toolset="web",
    schema={
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "Search the web for information",
            "parameters": { ... }
        }
    },
    handler=handle_web_search,
    check_fn=check_web_tools_available,
    requires_env=["OPENROUTER_API_KEY"],
)
```

### 工具集分组

工具按功能分组为"工具集"（Toolset），可以整体启用/禁用：

| 工具集 | 包含工具 | 说明 |
|--------|----------|------|
| `web` | web_search, web_extract | 网页搜索和内容提取 |
| `terminal` | execute_command | 终端命令执行 |
| `file` | read_file, write_file, patch_file | 文件读写 |
| `browser` | browser_navigate, browser_click, ... | 浏览器自动化 |
| `vision` | analyze_image | 图像分析 |
| `skills` | view_skill, list_skills | 技能管理 |
| `memory` | memory_read, memory_write | 记忆操作 |
| `cron` | create_cron, list_crons | 定时任务 |
| `delegate` | delegate_task | 子智能体委派 |

### 工具分发流程

```
LLM 返回 tool_call
    │
    ▼
model_tools.handle_function_call(name, args)
    │
    ▼
tools/registry.dispatch(name, args)
    │
    ▼
具体工具 handler 执行
    │
    ▼
返回结果 → 加入消息历史 → 继续循环
```

> 完整工具列表请参阅 [工具参考手册](./tools-reference.md)。

---

## 终端执行后端

终端工具支持多种执行环境，通过策略模式切换：

```
tools/environments/
├── base.py           # 抽象基类
├── local.py          # 本地 Shell（Bash/Zsh）
├── docker.py         # Docker 容器
├── ssh.py            # SSH 远程执行
├── modal.py          # Modal 无服务器计算
├── managed_modal.py  # 带状态的 Modal
├── daytona.py        # Daytona 云 IDE
├── singularity.py    # Singularity 容器
└── persistent_shell.py  # 长连接 Shell 会话
```

通过 `config.yaml` 中的 `terminal.env_type` 切换：

```yaml
terminal:
  env_type: local      # 或 docker, ssh, modal, daytona
  docker_image: ubuntu:22.04
  ssh_host: my-server.com
```

---

## 消息网关架构

网关负责将 Hermes Agent 连接到各种聊天平台：

```
用户（Telegram/Discord/Slack/...）
    │
    ▼
gateway/platforms/telegram.py（平台适配器）
    │
    ▼
gateway/run.py（GatewayRunner 统一编排）
    │
    ├── 创建/恢复 Session
    ├── 调用 AIAgent 处理消息
    ├── 流式返回响应到平台
    └── 处理审批工作流（危险命令确认）
```

> 详细配置请参阅 [消息网关指南](./gateway-guide.md)。

---

## 设计模式总结

| 模式 | 使用位置 | 说明 |
|------|----------|------|
| **注册表模式** | `tools/registry.py` | 工具自注册，无需硬编码 |
| **策略模式** | `tools/environments/` | 可插拔的终端执行后端 |
| **插件模式** | `plugins/memory/` | 动态加载记忆提供商 |
| **网关模式** | `gateway/platforms/` | 统一的平台无关会话管理 |
| **无状态函数** | `agent/prompt_builder.py` | 提示词组装无副作用 |
| **迭代预算** | `IterationBudget` | 防止无限工具调用循环 |
| **原子写入** | `utils.py` | JSON/YAML 的崩溃安全写入 |

---

## 线程安全与错误处理

- `IterationBudget` 使用锁保护计数器
- 工具注册表设计为线程安全
- 异步工具通过每线程事件循环桥接
- JSON/YAML 写入使用原子操作（先写临时文件再重命名）
- LLM 调用使用 tenacity 做指数退避重试
- 上下文压缩失败后有 600 秒冷却期
- 平台断连后自动重连

---

## 安全机制

- **提示词注入检测** — `prompt_builder.py` 扫描上下文文件中的恶意指令
- **PII 脱敏** — `agent/redact.py` 对敏感信息进行遮蔽
- **命令审批** — `tools/approval.py` 对危险命令要求用户确认
- **DM 配对** — 防止未授权用户通过消息平台访问
- **凭证隔离** — 使用独立的环境变量管理 API 密钥
