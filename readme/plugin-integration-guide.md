# Hermes Plugin 集成指南（开发者）

本文档面向要在 Hermes Agent 中接入**自定义 plugin / provider / context engine** 的开发者。目标不是介绍“怎么用插件”，而是说明**代码里插件是怎么被发现、加载、暴露到 agent 运行时的**，以及你应该选哪种扩展方式。

这份文档基于当前仓库实现整理，覆盖：

- 通用 plugin：自定义工具、hook、项目级扩展
- Memory provider：外部记忆后端
- Context engine：替换上下文压缩/管理策略

如果你是第一次进入仓库，不建议直接从这篇开始。更好的顺序是：

1. [simple.md](./simple.md)
2. [development-guide.md](./development-guide.md)
3. [architecture.md](./architecture.md)
4. 再回来读这篇

相关背景建议先看：

- [系统架构](./architecture.md)
- [开发者指南](./development-guide.md)
- [工具参考手册](./tools-reference.md)
- [配置参考](./configuration-reference.md)

---

## 1. 先选扩展方式

Hermes 里“plugin”现在其实有 3 条不同的接入路径：

| 类型 | 适合场景 | 典型能力 | 放在哪里 | 如何启用 |
|------|----------|----------|-----------|----------|
| **通用 plugin** | 想在不改核心代码的前提下扩展工具/Hook | 注册 tool、hook、项目级逻辑 | `~/.hermes/plugins/<name>/`、`./.hermes/plugins/<name>/`、pip entry point | 自动发现；可通过 `plugins.disabled` 关闭 |
| **Memory provider** | 想接入新的长期记忆后端 | 预取、写回、额外 memory tools | `plugins/memory/<name>/` | `config.yaml` 里 `memory.provider: "<name>"` |
| **Context engine** | 想替换默认 ContextCompressor | 自定义压缩、状态跟踪、额外上下文工具 | `plugins/context_engine/<name>/`，或通用 plugin 注册 | `config.yaml` 里 `context.engine: "<name>"` |

一句话判断：

- 只是想给模型加几个工具或埋一些生命周期 hook，用**通用 plugin**。
- 想做“跨 session 记忆系统”，用**Memory provider**。
- 想替换“上下文快满了怎么办”的策略，用**Context engine**。

如果你的需求只是给 Hermes 增加一个固定内置能力，而且你愿意改核心仓库，**直接加 built-in tool** 往往比 plugin 更合适：实现放 `tools/`，导入加到 `model_tools.py`，工具集注册到 `toolsets.py`。

---

## 2. 通用 Plugin：最灵活的扩展点

### 2.1 运行时是怎么加载的

通用 plugin 的核心实现是 [`hermes_cli/plugins.py`](/Users/xuji/Code/ai/hermes-agent/hermes_cli/plugins.py)。

加载入口在 [`model_tools.py`](/Users/xuji/Code/ai/hermes-agent/model_tools.py)：

1. 内置 `tools/*.py` 先被导入并自注册到 `tools.registry`
2. MCP 工具再做一次 discover
3. 最后调用 `discover_plugins()` 发现并加载通用 plugin

发现顺序：

1. `~/.hermes/plugins/<name>/`
2. `./.hermes/plugins/<name>/`
条件：必须显式设置 `HERMES_ENABLE_PROJECT_PLUGINS=true`
3. pip entry point：`hermes_agent.plugins`

被关闭的 plugin 会记录在配置里：

```yaml
plugins:
  disabled:
    - noisy-plugin
```

### 2.2 最小目录结构

```text
~/.hermes/plugins/my_plugin/
├── plugin.yaml
├── __init__.py
├── schemas.py        # 可选，建议拆开
├── tools.py          # 可选，建议拆开
└── data/             # 可选，静态资源
```

最小要求只有两个：

- `plugin.yaml`
- `__init__.py`，并提供 `register(ctx)` 函数

### 2.3 `plugin.yaml` 建议字段

`PluginManager` 目前会读取这些 manifest 字段：

```yaml
name: my_plugin
version: "1.0.0"
description: "Project-specific tools and hooks"
author: "Your Team"
provides_tools:
  - my_plugin_search
provides_hooks:
  - pre_llm_call
requires_env:
  - MY_PLUGIN_API_KEY
```

注意 3 个实现细节：

1. `provides_tools` / `provides_hooks` 目前主要是**元数据**，不参与运行时强校验。
2. `requires_env` 目前主要被 `hermes plugins install` 用来提示用户补环境变量；**加载时不会自动阻止 plugin 被导入**。
3. 如果你真的需要按环境决定工具是否可见，应该在 `ctx.register_tool(..., check_fn=...)` 里自己做。

### 2.4 `register(ctx)` 能做什么

`register(ctx)` 收到的是 `PluginContext`，当前支持：

- `ctx.register_tool(...)`
- `ctx.register_hook(...)`
- `ctx.register_context_engine(...)`
- `ctx.register_cli_command(...)`
- `ctx.inject_message(...)`

最常用的是 `register_tool` 和 `register_hook`。

### 2.5 注册自定义工具

推荐结构：

```python
import json


MY_TOOL_SCHEMA = {
    "name": "my_plugin_echo",
    "description": "Echo back the provided text for debugging or diagnostics.",
    "parameters": {
        "type": "object",
        "properties": {
            "text": {"type": "string", "description": "Text to echo"}
        },
        "required": ["text"],
    },
}


def handle_echo(args: dict, **kwargs) -> str:
    text = args.get("text", "")
    return json.dumps({"echo": text}, ensure_ascii=False)


def register(ctx):
    ctx.register_tool(
        name="my_plugin_echo",
        toolset="plugin_my_plugin",
        schema=MY_TOOL_SCHEMA,
        handler=handle_echo,
        description="Echo helper",
        emoji="🔌",
    )
```

工具 handler 的约束和内置工具完全一样：

- 签名：`def handler(args: dict, **kwargs) -> str`
- 返回值：**必须是 JSON 字符串**
- 不要把异常直接抛出去，最好自己兜底后返回错误 JSON

建议：

- `toolset` 用独立命名空间，比如 `plugin_<plugin_name>`，避免和内置 toolset 冲突
- schema 的 `description` 直接决定模型会不会调用它，写得越具体越好

### 2.6 Plugin toolset 如何进入 `hermes tools`

通用 plugin 注册的工具会先进全局 `tools.registry`，随后：

- `toolsets.py` 会把“注册表里存在但静态 `TOOLSETS` 里没有”的 `toolset` 识别成**plugin toolset**
- `hermes tools` / `hermes setup tools` 会把这些 plugin toolset 展示出来
- `get_tool_definitions()` 在启用对应 toolset 时才会把这些工具暴露给模型

所以通用 plugin 的工具行为和内置工具很像：

- 能出现在 tool list 里
- 能被平台 toolset 开关控制
- 能参与 `enabled_toolsets` / `disabled_toolsets`

### 2.7 注册 Hook

当前 `VALID_HOOKS` 包括：

- `pre_tool_call`
- `post_tool_call`
- `pre_llm_call`
- `post_llm_call`
- `pre_api_request`
- `post_api_request`
- `on_session_start`
- `on_session_end`
- `on_session_finalize`
- `on_session_reset`

最重要的是 `pre_llm_call`：

- 这是**唯一一个返回值会被运行时消费**的 hook
- 你可以返回 `{"context": "..."}` 或纯字符串
- Hermes 会把这段内容**拼到当前 user message 上**
- 它不会改 system prompt，这样可以保持 prompt cache 稳定

适合做：

- RAG recall
- 动态策略注入
- 会话前置 guardrails

### 2.8 `inject_message()` 适合什么

`ctx.inject_message()` 只在 **CLI 模式** 有效：

- agent 空闲：作为下一条用户消息排队
- agent 正在执行：作为中断消息插入

如果在 gateway 模式调用，没有 CLI 引用，会直接返回 `False`。

### 2.9 项目级 plugin

如果你想把 plugin 跟仓库一起提交，而不是装到用户家目录：

```text
<repo>/.hermes/plugins/my_plugin/
```

但默认**不会加载**，必须显式开启：

```bash
export HERMES_ENABLE_PROJECT_PLUGINS=true
```

这是一个安全边界，避免用户进入一个不可信仓库时自动执行项目内插件代码。

### 2.10 pip 分发

如果想把 plugin 做成独立 Python 包，入口点写到 `pyproject.toml`：

```toml
[project.entry-points."hermes_agent.plugins"]
my-plugin = "my_plugin_package"
```

这样安装包后，下次 Hermes 启动会自动发现。

---

## 3. Memory Provider：做“外部记忆系统”应该走这条路

### 3.1 什么时候不要用通用 plugin，而要用 Memory provider

如果你的能力具备下面任意一个特征，应该优先做 Memory provider：

- 需要跨 session 的 recall / persistence
- 需要在每轮前自动 prefetch、每轮后自动 sync
- 需要在上下文压缩前提取长期信息
- 需要向 agent 暴露一组“记忆后端专属工具”

这套机制的核心是：

- 抽象接口：[`agent/memory_provider.py`](/Users/xuji/Code/ai/hermes-agent/agent/memory_provider.py)
- 统一编排：[`agent/memory_manager.py`](/Users/xuji/Code/ai/hermes-agent/agent/memory_manager.py)
- 插件发现：[`plugins/memory/__init__.py`](/Users/xuji/Code/ai/hermes-agent/plugins/memory/__init__.py)

### 3.2 目录结构

仓库内 provider 的约定目录是：

```text
plugins/memory/my_memory/
├── __init__.py
├── plugin.yaml
├── cli.py              # 可选
└── other_modules.py    # 可选
```

这里和通用 plugin 不一样：

- Memory provider 是**仓库内 provider 插件**
- 不是放 `~/.hermes/plugins/`
- 由 `memory.provider` 单选启用

### 3.3 最小实现方式

实现一个 `MemoryProvider` 子类：

```python
from agent.memory_provider import MemoryProvider


class MyMemoryProvider(MemoryProvider):
    @property
    def name(self) -> str:
        return "my_memory"

    def is_available(self) -> bool:
        return True

    def initialize(self, session_id: str, **kwargs) -> None:
        pass

    def get_tool_schemas(self) -> list[dict]:
        return []
```

再在 `__init__.py` 里提供一种注册方式：

```python
def register(ctx):
    ctx.register_memory_provider(MyMemoryProvider())
```

或者直接暴露一个 `MemoryProvider` 子类，加载器也会自动尝试实例化。

### 3.4 必须理解的运行时语义

`run_agent.py` 初始化时会：

1. 读 `config.yaml` 的 `memory.provider`
2. 从 `plugins/memory/<name>/` 加载 provider
3. 创建 `MemoryManager`
4. `initialize_all(...)`
5. 将 provider 的工具 schema 直接注入到 `self.tools`

几个关键事实：

1. **built-in memory 永远存在**，外部 provider 是 additive，不会把内置 MEMORY.md/USER.md 关掉。
2. **外部 provider 只能有一个**，`MemoryManager` 会拒绝第二个 external provider。
3. provider tools **不走** `tools.registry` / `toolsets.py`，而是由 `MemoryManager.handle_tool_call()` 单独路由。
4. 所以 provider 工具也**不会出现在**普通 plugin toolset 里。

### 3.5 应该实现哪些方法

优先级最高的接口：

- `name`
- `is_available()`
- `initialize()`
- `get_tool_schemas()`
- `handle_tool_call()`（如果你暴露了工具）

常用可选接口：

- `system_prompt_block()`
- `prefetch()`
- `queue_prefetch()`
- `sync_turn()`
- `on_turn_start()`
- `on_pre_compress()`
- `on_memory_write()`
- `on_delegation()`
- `on_session_end()`
- `shutdown()`

如果你要做 provider 的交互式配置，还应该实现：

- `get_config_schema()`
- `save_config()`
- `post_setup()`

### 3.6 配置与 CLI

启用方式：

```yaml
memory:
  provider: "my_memory"
```

Memory provider 的 CLI 扩展有**特殊约定**：

- 在 `plugins/memory/<name>/cli.py` 里提供 `register_cli(subparser)`
- `hermes_cli/main.py` 会只为**当前激活的 provider** 自动注入这组命令

这和通用 plugin 的 CLI 注册方式不同。

### 3.7 什么时候适合放到 `plugins/memory/`

如果你准备把这个能力作为 Hermes 仓库的一部分长期维护，放这里最合适。已有例子可以参考：

- `plugins/memory/honcho/`
- `plugins/memory/hindsight/`
- `plugins/memory/retaindb/`

---

## 4. Context Engine：替换上下文管理策略

### 4.1 适用场景

当你不是想“多记一点东西”，而是想替换 Hermes 在长对话时的**上下文压缩/摘要/工具化检索**策略时，用 Context engine。

核心接口在：

- [`agent/context_engine.py`](/Users/xuji/Code/ai/hermes-agent/agent/context_engine.py)

### 4.2 两种接入方式

当前代码支持两种来源：

1. 仓库内 `plugins/context_engine/<name>/`
2. 通用 plugin 里 `ctx.register_context_engine(engine)`

`run_agent.py` 的选择顺序是：

1. 读 `config.yaml` 的 `context.engine`
2. 先尝试 `plugins/context_engine/<name>/`
3. 再尝试通用 plugin 注册出来的 engine
4. 都没有就回退到内置 `ContextCompressor`

### 4.3 最小实现

```python
from agent.context_engine import ContextEngine


class MyEngine(ContextEngine):
    @property
    def name(self) -> str:
        return "my_engine"

    def update_from_response(self, usage: dict) -> None:
        self.last_prompt_tokens = usage.get("prompt_tokens", 0)
        self.last_completion_tokens = usage.get("completion_tokens", 0)
        self.last_total_tokens = usage.get("total_tokens", 0)

    def should_compress(self, prompt_tokens: int = None) -> bool:
        tokens = prompt_tokens if prompt_tokens is not None else self.last_prompt_tokens
        return tokens >= self.threshold_tokens

    def compress(self, messages: list[dict], current_tokens: int = None) -> list[dict]:
        return messages
```

如果放在通用 plugin 里：

```python
def register(ctx):
    ctx.register_context_engine(MyEngine())
```

### 4.4 额外工具

Context engine 可以暴露自己的工具：

- `get_tool_schemas()`
- `handle_tool_call()`

运行时会把这些 schema 直接 append 到 `self.tools`，和 memory provider 一样，**不经过普通 toolset 过滤**。

这意味着：

- 只要 engine 被激活，这些工具就对模型可见
- tool call 路由直接进 `self.context_compressor.handle_tool_call(...)`

### 4.5 配置

```yaml
context:
  engine: "my_engine"
```

如果值是 `"compressor"`，则明确使用内置 `ContextCompressor`，不会自动切到插件引擎。

---

## 5. 当前工程里和 Plugin 相关的关键实现点

### 5.1 通用 plugin 的真实入口

- 发现/加载：[`hermes_cli/plugins.py`](/Users/xuji/Code/ai/hermes-agent/hermes_cli/plugins.py)
- 安装/更新/禁用：[`hermes_cli/plugins_cmd.py`](/Users/xuji/Code/ai/hermes-agent/hermes_cli/plugins_cmd.py)
- tool surface 合并：[`model_tools.py`](/Users/xuji/Code/ai/hermes-agent/model_tools.py)

### 5.2 Memory provider 的真实入口

- provider 接口：[`agent/memory_provider.py`](/Users/xuji/Code/ai/hermes-agent/agent/memory_provider.py)
- manager：[`agent/memory_manager.py`](/Users/xuji/Code/ai/hermes-agent/agent/memory_manager.py)
- repo 内 provider 发现：[`plugins/memory/__init__.py`](/Users/xuji/Code/ai/hermes-agent/plugins/memory/__init__.py)
- agent 初始化接线：[`run_agent.py`](/Users/xuji/Code/ai/hermes-agent/run_agent.py)

### 5.3 Context engine 的真实入口

- engine 接口：[`agent/context_engine.py`](/Users/xuji/Code/ai/hermes-agent/agent/context_engine.py)
- repo 内 engine 发现：[`plugins/context_engine/__init__.py`](/Users/xuji/Code/ai/hermes-agent/plugins/context_engine/__init__.py)
- agent 初始化接线：[`run_agent.py`](/Users/xuji/Code/ai/hermes-agent/run_agent.py)

---

## 6. 配置示例

### 6.1 开启一个 memory provider

```yaml
memory:
  provider: "honcho"
```

### 6.2 开启一个 context engine

```yaml
context:
  engine: "lcm"
```

### 6.3 禁用某个通用 plugin

```yaml
plugins:
  disabled:
    - my_plugin
```

### 6.4 项目级 plugin 开关

```bash
export HERMES_ENABLE_PROJECT_PLUGINS=true
```

---

## 7. 开发建议与坑

### 7.1 `requires_env` 不等于运行时 gating

当前仓库里：

- `requires_env` 会在 `hermes plugins install` 时用来提示补变量
- 但 `PluginManager` 加载 plugin 时**不会自动因为缺环境变量而跳过**

如果你需要真实的运行时 gating，请自己做：

- tool 级别：`check_fn=...`
- plugin 内部：在 `register()` 或 handler 里显式检查

### 7.2 通用 plugin CLI 注册接口已存在，但主 CLI 目前没有自动接线

`PluginContext.register_cli_command()` 已经实现，也有测试；但当前 [`hermes_cli/main.py`](/Users/xuji/Code/ai/hermes-agent/hermes_cli/main.py) 自动接入的是：

- `plugins/memory/<name>/cli.py` 的 `register_cli(subparser)`

也就是说，**通用 plugin 的 CLI command 注册能力目前更像“预留接口”**。如果你准备在仓库里真正依赖这套能力，需要同时把 `get_plugin_cli_commands()` 接到主 argparse 构建流程里。

### 7.3 schema 描述要写清楚

模型是否调用你的工具，主要看 schema：

- 做什么
- 什么时候该调用
- 参数语义是什么

不要写成“does stuff”这种抽象描述。

### 7.4 返回值必须是 JSON 字符串

这是 Hermes 整个工具体系的统一约束，plugin tool / memory tool / context engine tool 都一样。

### 7.5 profile-safe 路径

如果 plugin/provider 要写状态文件，按仓库约定应该走 `HERMES_HOME` 作用域：

```python
from hermes_constants import get_hermes_home

state_dir = get_hermes_home() / "my_plugin"
```

不要硬编码 `~/.hermes`，否则会破坏 profile 隔离。

### 7.6 Memory provider / Context engine 都是单选

不要按“多个 provider 并行运行”的思路设计：

- external memory provider：最多 1 个
- context engine：最多 1 个

这是运行时约束，不是 UI 约束。

---

## 8. 测试与调试

先激活环境：

```bash
source venv/bin/activate
```

建议测试命令：

```bash
python -m pytest tests/hermes_cli/test_plugins.py -q
python -m pytest tests/hermes_cli/test_plugin_cli_registration.py -q
python -m pytest tests/agent/test_memory_provider.py -q
python -m pytest tests/agent/test_context_engine.py -q
python -m pytest tests/plugins/ -q
```

调试时常用检查点：

```python
from hermes_cli.plugins import get_plugin_manager
from tools.registry import registry

mgr = get_plugin_manager()
print(mgr.list_plugins())
print(sorted(registry.get_tool_to_toolset_map().items()))
```

如果要确认工具是否真正暴露给模型，可以直接看：

```python
from model_tools import get_tool_definitions

tools = get_tool_definitions(enabled_toolsets=["plugin_my_plugin"], quiet_mode=True)
print([t["function"]["name"] for t in tools])
```

---

## 9. 给开发者的实操建议

如果你是在这个仓库里做二次开发，通常按下面决策就够了：

1. 只是扩一个工具或 hook，而且不想改核心目录：做通用 plugin。
2. 想做可复用、可配置、可在所有 session 生效的记忆后端：做 `plugins/memory/<name>/`。
3. 想替换 context compression / DAG / recall 策略：做 context engine。
4. 想把能力直接并入 Hermes 主产品、参与原生 toolset 管理：不要做 plugin，直接加 `tools/*.py`。

如果你只打算在一个仓库里本地使用，优先用：

- `./.hermes/plugins/<name>/` + `HERMES_ENABLE_PROJECT_PLUGINS=true`

如果你想让能力成为 Hermes 仓库的一部分，优先用：

- `plugins/memory/<name>/`
- `plugins/context_engine/<name>/`
- 或直接内置到 `tools/`

如果你想把能力独立发布给别人安装，优先用：

- `~/.hermes/plugins/<name>/`
- 或 pip entry point
