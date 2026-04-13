# Hermes Agent 定制开发指南

## 开发环境

```bash
git clone <repo-url> hermes-agent && cd hermes-agent
uv venv venv --python 3.11 && source venv/bin/activate
uv pip install -e ".[all,dev]"
hermes doctor  # 验证
```

---

## 架构概要

核心是一个工具调用循环：用户输入 -> 构建 prompt -> 调用 LLM -> 解析响应 -> 若有 tool_calls 则执行工具并回到 LLM，否则返回最终回答。

```
run_agent.py (AIAgent)          # 核心编排：对话循环、工具分发、会话管理
├── agent/
│   ├── prompt_builder.py       # 系统提示词组装（身份/平台/技能索引/记忆）
│   ├── context_compressor.py   # 上下文压缩（接近 token 上限时触发）
│   ├── model_metadata.py       # 模型元数据（context length、定价）
│   ├── memory_manager.py       # 记忆管理（预取/同步）
│   └── skill_utils.py          # 技能元数据解析
├── model_tools.py              # 工具发现 & 分发
├── tools/registry.py           # 工具注册中心
├── tools/                      # 50+ 工具实现
├── toolsets.py                 # 工具集分组定义
├── gateway/                    # 消息平台网关
│   ├── run.py                  # GatewayRunner
│   ├── session.py              # 会话管理
│   ├── delivery.py             # 消息投递
│   └── platforms/              # 各平台适配器
├── plugins/memory/             # 记忆提供商插件
├── cli.py                      # 交互式 TUI
├── hermes_cli/                 # CLI 子命令
├── skills/                     # 内置技能（Markdown）
├── cron/                       # 定时任务调度
└── acp_adapter/                # 编辑器集成（VS Code、Cursor）
```

### 快速定位

| 想改什么 | 去哪里 |
|---------|--------|
| 核心对话循环 | `run_agent.py` → `AIAgent` |
| 系统提示词 | `agent/prompt_builder.py` |
| 上下文压缩 | `agent/context_compressor.py` |
| 工具注册机制 | `tools/registry.py` |
| 工具集分组 | `toolsets.py`、`toolset_distributions.py` |
| 工具发现/分发 | `model_tools.py` |
| 某个工具的实现 | `tools/<tool_name>.py` |
| 某个平台网关 | `gateway/platforms/<platform>.py` |
| 网关会话管理 | `gateway/session.py` |
| 技能解析 | `agent/skill_utils.py` |
| 定时任务 | `cron/scheduler.py`、`cron/jobs.py` |

### 核心设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 注册表 | `tools/registry.py` | 工具自注册，import 即注册 |
| 策略 | `tools/environments/` | 可插拔的终端执行后端（local/docker/ssh/modal...） |
| 插件 | `plugins/memory/` | 动态加载记忆提供商 |
| 网关适配 | `gateway/platforms/` | 统一接口，平台无关 |

---

## 扩展点

### 1. 添加工具

**Step 1** — 在 `tools/` 下创建文件：

```python
# tools/my_tool.py
from tools.registry import registry

def check_available():
    import os
    return os.environ.get("MY_API_KEY") is not None

async def handle_my_action(arguments: dict, **kwargs) -> str:
    param1 = arguments.get("param1", "")
    return f"结果: {param1}"

registry.register(
    name="my_action",
    toolset="my_toolset",
    schema={
        "type": "function",
        "function": {
            "name": "my_action",
            "description": "执行我的操作",
            "parameters": {
                "type": "object",
                "properties": {
                    "param1": {"type": "string", "description": "参数说明"}
                },
                "required": ["param1"]
            }
        }
    },
    handler=handle_my_action,
    check_fn=check_available,
    requires_env=["MY_API_KEY"],
    is_async=True,
)
```

**Step 2** — 在 `model_tools.py` 的 `_discover_tools()` 中添加 `from tools import my_tool`

**Step 3**（可选）— 若创建了新工具集，在 `toolsets.py` 中注册：`TOOLSETS["my_toolset"] = ["my_action"]`

要点：
- handler 返回字符串（成功失败都是），不要抛出未捕获异常
- `check_fn` 控制工具是否对 LLM 可见
- `requires_env` 声明依赖的环境变量

### 2. 添加消息平台

在 `gateway/platforms/` 下创建适配器，继承 `base.py` 中的抽象基类：

```python
# gateway/platforms/my_platform.py
from gateway.platforms.base import BasePlatform

class MyPlatformAdapter(BasePlatform):
    async def start(self):
        """启动连接"""
    async def stop(self):
        """断开连接"""
    async def send_message(self, chat_id: str, text: str, **kwargs):
        """发送消息"""
    async def on_message(self, message):
        """收到消息 -> 转为统一格式交给 GatewayRunner"""
```

然后在 `gateway/config.py` 的 `Platform` 枚举中注册。

### 3. 添加记忆提供商

```python
# plugins/memory/my_memory/__init__.py
def register(ctx):
    from agent.memory_provider import MemoryProvider

    class MyMemoryProvider(MemoryProvider):
        async def prefetch(self, user_message: str) -> str:
            """对话开始前预取相关记忆"""
        async def sync(self, user_msg: str, assistant_response: str):
            """对话结束后持久化"""

    ctx.add_provider(MyMemoryProvider())
```

通过 `config.yaml` 中 `memory.provider` 切换。

### 4. 创建技能

技能是 Markdown 增强提示词，放在 `~/.hermes/skills/<类别>/<技能名>/SKILL.md`：

```markdown
---
platforms: [macos, linux]
tags: [automation]
requires: [GITHUB_TOKEN]
---

# 技能名称

简短描述（第一段作为索引摘要）。

## 步骤

1. ...
2. ...
```

技能会在 prompt 构建时被扫描索引，匹配后注入系统提示词。相关代码：`agent/skill_utils.py`、`agent/prompt_builder.py` 中的 `build_skills_system_prompt()`。

### 5. MCP 工具集成

在 `~/.hermes/config.yaml` 中声明 MCP 服务器，工具会自动注册：

```yaml
mcp:
  servers:
    - name: my-server
      command: npx
      args: ["-y", "@my/mcp-server"]
      env:
        API_KEY: "..."
```

### 6. 终端执行后端

通过策略模式切换（`tools/environments/`），实现 `base.py` 的抽象基类即可添加新后端：

```
tools/environments/
├── base.py            # 抽象基类
├── local.py           # 本地 Shell
├── docker.py          # Docker
├── ssh.py             # SSH
├── modal.py           # Modal
├── daytona.py         # Daytona
├── singularity.py     # Singularity
└── persistent_shell.py
```

通过 `config.yaml` 中 `terminal.env_type` 切换。

---

## 配置体系

配置目录：`~/.hermes/`（可通过 `HERMES_HOME` 环境变量覆盖）

```
~/.hermes/
├── config.yaml     # 主配置
├── .env            # API 密钥
├── skills/         # 技能
├── memories/       # 记忆
├── state.db        # 会话数据库
├── cron/           # 定时任务
└── profiles/       # 多配置隔离
```

关键配置项：

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: "openrouter"       # openrouter/anthropic/openai/custom
  base_url: null               # 自定义端点

terminal:
  env_type: "local"            # local/docker/ssh/modal/daytona/singularity

compression:
  enabled: true
  threshold: 0.5               # token 使用率阈值

agent:
  max_turns: 50
  system_prompt: null           # 追加自定义提示词

delegation:
  max_iterations: 20
  default_toolsets: [terminal, file, web]

toolsets:
  enabled: [web, terminal, file, skills, memory, cron, delegate]
  disabled: [browser, vision]

skills:
  disabled: [social-media/xitter]
  external_dirs: [/path/to/custom/skills]

memory:
  provider: "builtin"          # builtin/mem0/honcho/...
```

环境变量覆盖：`HERMES_MODEL`、`HERMES_BASE_URL`、`HERMES_PROVIDER`、`HERMES_PROFILE`

---

## 测试

```bash
pytest tests/ -v                          # 全部
pytest tests/test_agent_loop.py -v        # 单文件
pytest tests/test_agent_loop.py::test_name -v  # 单测试
pytest tests/ --cov=. --cov-report=html   # 覆盖率
```

测试目录与源码对应：`tests/agent/`、`tests/gateway/`、`tests/hermes_cli/`、`tests/plugins/` 等。需要 API 的工具用 `unittest.mock` 模拟。

---

## 调试

```bash
HERMES_LOG_LEVEL=DEBUG hermes   # 详细日志
HERMES_LOG_API=1 hermes         # 查看 API 请求
```

```python
# 检查工具注册
from tools.registry import registry
print(registry.get_tool_to_toolset_map())

# 直接调用工具
from model_tools import handle_function_call
result = handle_function_call("web_search", {"query": "test"})
```

---

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `openai` | LLM API（所有提供商走 OpenAI 兼容接口） |
| `anthropic` | Anthropic 专用功能（prompt caching 等） |
| `httpx` | 异步 HTTP |
| `prompt_toolkit` | TUI |
| `rich` | 终端美化 |
| `pydantic` | 数据验证 |
| `tenacity` | API 调用重试 |
| `jinja2` | 提示词模板 |

## 代码约定

- Python 3.11+ 类型注解
- `async/await` 处理 I/O
- 工具 handler 返回字符串，不抛异常
- 每个工具一个文件，自注册
- 使用 `hermes_logging` 记录日志
- JSON/YAML 原子写入（先写临时文件再 rename）
