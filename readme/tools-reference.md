# 工具参考手册

Hermes 的工具系统是整个 agent 的“可执行能力层”。模型不是直接调用任意 Python 函数，而是只能调用那些被注册到 `tools.registry`、并且在当前运行上下文中可见的工具。

---

## 新人先知道 3 件事

1. **Tool 是单个能力**
例如 `web_search`、`read_file`、`terminal`
2. **Toolset 是工具分组**
例如 `web`、`file`、`terminal`
3. **工具文件存在，不代表模型就能看到**
还要同时满足：已注册、通过 `check_fn`、当前 toolset 已启用

排查“工具为什么没出来”时，不要只看 `tools/<name>.py`，还要看：

- `tools/registry.py`
- `model_tools.py`
- `toolsets.py`
- 当前配置中的 toolset

---

## 工具注册机制

每个工具模块在 import 时调用 `registry.register(...)`：

```python
registry.register(
    name="tool_name",
    toolset="toolset_name",
    schema={...},
    handler=handler_fn,
    check_fn=check_fn,
    requires_env=["API_KEY"],
    is_async=False,
    description="...",
    emoji="🔧",
)
```

运行流程大致是：

1. `model_tools.py` 导入各工具模块
2. 工具模块执行 `registry.register(...)`
3. `get_tool_definitions()` 按 toolset 和可用性过滤 schema
4. 模型看到 schema 后决定是否发起 tool call
5. `registry.dispatch()` 调用具体 handler

---

## 一个工具从代码到模型的完整链路

如果你想真正理解“为什么这个工具能被模型调用”，可以按这条链看：

1. 代码实现写在 `tools/<name>.py`
2. 模块 import 时执行 `registry.register(...)`
3. `model_tools.py` discover 到这个模块
4. `get_tool_definitions()` 按 toolset、可用性、平台过滤 schema
5. schema 进入本轮 API 请求
6. 模型返回 tool call
7. `handle_function_call()` / `registry.dispatch()` 把调用路由回 handler

这条链里任何一环出问题，都会表现成：

- 工具根本没出现在模型可见列表里
- 工具名存在，但调用时报错
- 工具在某个平台可见，在另一个入口不可见

---

## 当前常见工具名

下面是当前内置工具里最常见的一组名字：

- `web_search`
- `web_extract`
- `terminal`
- `process`
- `read_file`
- `write_file`
- `patch`
- `search_files`
- `browser_navigate`
- `browser_snapshot`
- `browser_click`
- `browser_type`
- `browser_scroll`
- `browser_back`
- `browser_press`
- `browser_get_images`
- `browser_vision`
- `browser_console`
- `vision_analyze`
- `skills_list`
- `skill_view`
- `skill_manage`
- `memory`
- `todo`
- `session_search`
- `clarify`
- `execute_code`
- `delegate_task`
- `cronjob`
- `send_message`
- `image_generate`
- `text_to_speech`

如果别的文档里出现旧名字，以当前实现和注册表为准。

---

## 核心 toolset 一览

### `web`

| 工具 | 文件 | 说明 |
|------|------|------|
| `web_search` | `tools/web_tools.py` | 搜索网页 |
| `web_extract` | `tools/web_tools.py` | 提取网页正文 |

### `terminal`

| 工具 | 文件 | 说明 |
|------|------|------|
| `terminal` | `tools/terminal_tool.py` | 执行终端命令 |
| `process` | `tools/process_registry.py` | 管理后台进程 |

### `file`

| 工具 | 文件 | 说明 |
|------|------|------|
| `read_file` | `tools/file_tools.py` | 读取文件 |
| `write_file` | `tools/file_tools.py` | 写文件 |
| `patch` | `tools/file_tools.py` | 局部修改文件 |
| `search_files` | `tools/file_tools.py` | 搜索文件名或内容 |

### `browser`

| 工具 | 文件 | 说明 |
|------|------|------|
| `browser_navigate` | `tools/browser_tool.py` | 打开 URL |
| `browser_snapshot` | `tools/browser_tool.py` | 获取页面快照 |
| `browser_click` | `tools/browser_tool.py` | 点击页面元素 |
| `browser_type` | `tools/browser_tool.py` | 输入文本 |
| `browser_scroll` | `tools/browser_tool.py` | 滚动页面 |
| `browser_back` | `tools/browser_tool.py` | 返回上一页 |
| `browser_press` | `tools/browser_tool.py` | 发送按键 |
| `browser_get_images` | `tools/browser_tool.py` | 获取页面图片 |
| `browser_vision` | `tools/browser_tool.py` | 对页面做视觉分析 |
| `browser_console` | `tools/browser_tool.py` | 读取控制台信息 |

### `vision`

| 工具 | 文件 | 说明 |
|------|------|------|
| `vision_analyze` | `tools/vision_tools.py` | 分析图片内容 |

### `skills`

| 工具 | 文件 | 说明 |
|------|------|------|
| `skills_list` | `tools/skills_tool.py` | 列出技能 |
| `skill_view` | `tools/skills_tool.py` | 查看技能 |
| `skill_manage` | `tools/skill_manager_tool.py` | 创建或更新技能 |

### `memory`

| 工具 | 文件 | 说明 |
|------|------|------|
| `memory` | `tools/memory_tool.py` | 读写内置持久化记忆 |

### `cronjob`

| 工具 | 文件 | 说明 |
|------|------|------|
| `cronjob` | `tools/cronjob_tools.py` | 创建、更新、暂停、恢复、执行定时任务 |

### `delegation`

| 工具 | 文件 | 说明 |
|------|------|------|
| `delegate_task` | `tools/delegate_tool.py` | 创建子智能体执行子任务 |

---

## 其他常用工具

| 工具 | 文件 | 说明 |
|------|------|------|
| `execute_code` | `tools/code_execution_tool.py` | 程序化调用工具 |
| `clarify` | `tools/clarify_tool.py` | 向用户提问澄清 |
| `todo` | `tools/todo_tool.py` | 跟踪多步任务 |
| `session_search` | `tools/session_search_tool.py` | 搜索历史会话 |
| `image_generate` | `tools/image_generation_tool.py` | 图像生成 |
| `text_to_speech` | `tools/tts_tool.py` | 文本转语音 |
| `send_message` | `tools/send_message_tool.py` | 发送消息到平台 |
| `ha_list_entities` / `ha_get_state` / `ha_list_services` / `ha_call_service` | `tools/homeassistant_tool.py` | Home Assistant 控制 |
| `mixture_of_agents` | `tools/mixture_of_agents_tool.py` | 多模型协作 |

---

## 启用和禁用工具

### 通过 CLI

```bash
hermes tools
hermes tools list --platform cli
```

### 通过配置

```yaml
toolsets:
  enabled:
    - web
    - terminal
    - file
  disabled:
    - browser
```

### 通过聊天命令

```text
/tools
/toolsets
```

---

## 调试“工具为什么不可用”

按这个顺序排查最省时间：

1. 工具模块是否在 `model_tools.py` 中被 discover
2. `registry.register(...)` 是否真的执行了
3. `check_fn` 是否返回 False
4. 当前 toolset 是否启用
5. 当前入口是不是 CLI / gateway / ACP，它们的默认 toolset 是否不同

如果你想直接看注册结果：

```python
from tools.registry import registry
print(registry.get_tool_to_toolset_map())
```

如果你想确认最终暴露给模型的是哪一批 schema，可以直接看：

```python
from model_tools import get_tool_definitions

tools = get_tool_definitions(enabled_toolsets=["web", "file", "terminal"], quiet_mode=True)
print([tool["function"]["name"] for tool in tools])
```

---

## 添加自定义工具

如果你要扩工具，请继续看 [development-guide.md](./development-guide.md)。

如果你不想改核心仓库，而是想通过运行时扩展加入能力，请看 [plugin-integration-guide.md](./plugin-integration-guide.md)。

---

## MCP 工具

MCP server 的工具会在运行时被发现并注册，因此它们和内置工具看起来很像，但来源不同。

典型配置：

```yaml
mcp:
  servers:
    - name: my-server
      command: npx
      args: ["-y", "@my/mcp-server"]
```

MCP 工具通常会出现在工具系统里，但调试时要记得把它和“仓库内置工具”区分开看。
