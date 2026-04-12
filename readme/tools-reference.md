# 工具参考手册

Hermes Agent 通过自注册的工具系统扩展自身能力。本文档列出所有内置工具及其所属工具集。

---

## 工具注册机制

每个工具文件在 `tools/` 目录下，于模块导入时自动调用 `registry.register()` 注册自己。核心注册中心是 `tools/registry.py`。

```python
# 注册一个工具的基本结构
registry.register(
    name="tool_name",          # 唯一名称
    toolset="toolset_name",    # 所属工具集
    schema={...},              # OpenAI 格式的 JSON Schema
    handler=handler_fn,        # 实际执行函数
    check_fn=check_fn,         # 可用性检查函数
    requires_env=["API_KEY"],  # 所需环境变量
    is_async=False,            # 是否异步
    description="...",         # 简短描述
    emoji="🔧",               # 显示用图标
)
```

---

## 工具集一览

### `web` — 网络工具

| 工具 | 文件 | 说明 |
|------|------|------|
| `web_search` | `tools/web_tools.py` | 搜索网页（Exa、Firecrawl） |
| `web_extract` | `tools/web_tools.py` | 提取网页正文内容 |

**所需环境变量：** `OPENROUTER_API_KEY` 或 `EXA_API_KEY`

### `terminal` — 终端执行

| 工具 | 文件 | 说明 |
|------|------|------|
| `execute_command` | `tools/terminal_tool.py` | 在配置的环境中执行 Shell 命令 |

**执行后端：** local / docker / ssh / modal / daytona / singularity

### `file` — 文件操作

| 工具 | 文件 | 说明 |
|------|------|------|
| `read_file` | `tools/file_tools.py` | 读取文件内容 |
| `write_file` | `tools/file_tools.py` | 写入/创建文件 |
| `patch_file` | `tools/file_tools.py` | 对文件进行局部修改（diff patch） |

### `browser` — 浏览器自动化

| 工具 | 文件 | 说明 |
|------|------|------|
| `browser_navigate` | `tools/browser_tool.py` | 打开 URL |
| `browser_click` | `tools/browser_tool.py` | 点击页面元素 |
| `browser_type` | `tools/browser_tool.py` | 在输入框中键入文字 |
| `browser_screenshot` | `tools/browser_tool.py` | 截取页面截图 |

**浏览器提供商：** Browser Use / Browserbase / Firecrawl / CamoFox（隐匿模式）

### `vision` — 视觉工具

| 工具 | 文件 | 说明 |
|------|------|------|
| `analyze_image` | `tools/vision_tools.py` | 分析图片内容 |

### `skills` — 技能管理

| 工具 | 文件 | 说明 |
|------|------|------|
| `view_skill` | `tools/skills_tool.py` | 查看技能内容 |
| `list_skills` | `tools/skills_tool.py` | 列出所有可用技能 |

### `skill_manager` — 技能创建

| 工具 | 文件 | 说明 |
|------|------|------|
| `create_skill` | `tools/skill_manager_tool.py` | 创建新技能 |
| `update_skill` | `tools/skill_manager_tool.py` | 更新已有技能 |

### `memory` — 记忆工具

| 工具 | 文件 | 说明 |
|------|------|------|
| `memory_read` | `tools/memory_tool.py` | 读取持久化记忆 |
| `memory_write` | `tools/memory_tool.py` | 写入持久化记忆 |
| `memory_search` | `tools/memory_tool.py` | 搜索记忆内容 |

### `cron` — 定时任务

| 工具 | 文件 | 说明 |
|------|------|------|
| `create_cron` | `tools/cronjob_tools.py` | 创建定时任务 |
| `list_crons` | `tools/cronjob_tools.py` | 列出所有定时任务 |
| `delete_cron` | `tools/cronjob_tools.py` | 删除定时任务 |

### `delegate` — 子智能体

| 工具 | 文件 | 说明 |
|------|------|------|
| `delegate_task` | `tools/delegate_tool.py` | 创建隔离的子智能体来执行子任务 |

### 其他独立工具

| 工具 | 文件 | 说明 |
|------|------|------|
| `execute_code` | `tools/execute_code.py` | Python 代码执行（RPC 风格） |
| `clarify` | `tools/clarify_tool.py` | 向用户提问澄清 |
| `text_to_speech` | `tools/tts_tool.py` | 文本转语音（Edge TTS / ElevenLabs） |
| `transcribe` | `tools/transcription_tools.py` | 语音转文字 |
| `generate_image` | `tools/image_generation_tool.py` | 图像生成（FAL） |
| `mixture_of_agents` | `tools/mixture_of_agents_tool.py` | 多模型路由 |
| `homeassistant` | `tools/homeassistant_tool.py` | 智能家居控制 |
| `send_message` | `tools/send_message_tool.py` | 主动发送消息到平台 |

---

## 工具集启用/禁用

### 通过 CLI

```bash
hermes tools           # 交互式管理
hermes toolsets        # 查看工具集状态
```

### 通过配置文件

```yaml
# ~/.hermes/config.yaml
toolsets:
  enabled:
    - web
    - terminal
    - file
    - skills
  disabled:
    - browser    # 禁用浏览器工具
```

### 通过聊天命令

```
/tools          # 查看当前工具
/toolsets       # 管理工具集
```

---

## 添加自定义工具

想添加自己的工具？参见 [开发者指南 — 添加新工具](./development-guide.md#添加新工具)。

核心步骤：

1. 在 `tools/` 下创建新文件
2. 定义 JSON Schema 和 handler 函数
3. 调用 `registry.register()` 注册
4. 在 `model_tools.py` 的导入列表中添加你的模块
5. 编写测试

---

## MCP 工具集成

Hermes 支持通过 MCP（Model Context Protocol）协议连接外部工具服务器：

```yaml
# ~/.hermes/config.yaml
mcp:
  servers:
    - name: my-server
      command: npx
      args: ["-y", "@my/mcp-server"]
```

MCP 工具会自动被发现并注册到工具系统中，与内置工具使用方式一致。
