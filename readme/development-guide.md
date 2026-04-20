# 开发者指南

这篇文档解决的是“我已经决定改 Hermes 了，接下来应该怎么做”。

如果你是第一次进入仓库，建议先读 [simple.md](./simple.md)。

---

## 新人先做这 4 步

1. 激活环境：`source venv/bin/activate`
2. 跑 `hermes doctor`
3. 跑一次 `hermes`
4. 跑一个最小测试：`python -m pytest tests/hermes_cli/ -q`

这样能先确认：

- 依赖没坏
- CLI 可用
- 测试框架可运行

---

## 开发环境搭建

### 1. 克隆并安装

```bash
git clone <repo-url> hermes-agent
cd hermes-agent

uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"

# 可选：RL 训练子模块
git submodule update --init tinker-atropos
uv pip install -e "./tinker-atropos"

# 可选：浏览器和 bridge 相关依赖
npm install
```

### 2. 验证安装

```bash
hermes doctor
python -m pytest tests/ -v --co
```

---

## 推荐阅读顺序

不要一上来就从 `run_agent.py` 第 1 行读到最后。更高效的顺序是：

1. [simple.md](./simple.md)
2. [architecture.md](./architecture.md)
3. `run_agent.py` 里只看 `AIAgent.__init__` 和 `run_conversation()`
4. `model_tools.py`
5. `tools/registry.py`

如果你只改一个局部功能，就按目标找文件，不要试图第一次就完整建模全仓库。

---

## 测试

### 常用命令

```bash
# 全量
python -m pytest tests/ -v

# 单文件
python -m pytest tests/test_agent_loop.py -v

# 单测试
python -m pytest tests/test_agent_loop.py::test_name -v

# 按目录
python -m pytest tests/gateway/ -v
python -m pytest tests/hermes_cli/ -v
python -m pytest tests/agent/ -v
python -m pytest tests/plugins/ -v

# 覆盖率
python -m pytest tests/ --cov=. --cov-report=html
```

### 经验法则

- 小改动先跑**最小相关测试**
- 改 CLI 跑 `tests/hermes_cli/`
- 改 gateway 跑 `tests/gateway/`
- 改 provider / plugin 跑 `tests/plugins/`
- 改 agent 主循环再考虑跑更大范围

### 测试目录结构

```text
tests/
├── agent/            # agent 内部模块
├── gateway/          # 消息网关
├── hermes_cli/       # CLI 子命令与交互
├── plugins/          # plugin / provider / engine
├── cron/             # 定时任务
├── integration/      # 集成测试
└── e2e/              # 端到端测试
```

---

## 代码结构速查

| 我想改什么 | 去哪里 |
|------------|---------|
| 主对话循环 | `run_agent.py` |
| 系统提示词组装 | `agent/prompt_builder.py` |
| 上下文压缩 | `agent/context_compressor.py` |
| 工具注册机制 | `tools/registry.py` |
| 工具 discover / dispatch | `model_tools.py` |
| 具体工具实现 | `tools/<tool>.py` |
| toolset 定义 | `toolsets.py` |
| CLI 入口 | `cli.py` |
| CLI 子命令 | `hermes_cli/main.py`、`hermes_cli/commands.py` |
| 消息网关主入口 | `gateway/run.py` |
| 平台适配器 | `gateway/platforms/<platform>.py` |
| 技能元数据与技能命令 | `agent/skill_utils.py`、`agent/skill_commands.py` |
| plugin 系统 | `hermes_cli/plugins.py`、`plugins/` |

---

## 第一个改动建议怎么选

第一次提 PR，优先挑这些低风险改动：

1. 修文档和实现不一致
2. 给现有工具补测试
3. 修一个帮助文案或错误提示
4. 给配置项补注释或默认值说明
5. 修一个小型工具注册 / toolset 展示问题

不建议第一次就做：

- 大规模拆 `run_agent.py`
- 同时碰 CLI、gateway、tool registry 三层
- 动 fallback / provider / compression 的核心路径

---

## 添加新工具

这是最常见的扩展场景。

### 步骤 1：创建工具文件

```python
from tools.registry import registry
import json


def check_my_tool_available():
    import os
    return os.environ.get("MY_API_KEY") is not None


async def handle_my_action(arguments: dict, **kwargs) -> str:
    param1 = arguments.get("param1", "")
    return json.dumps({"result": f"处理了: {param1}"}, ensure_ascii=False)


registry.register(
    name="my_action",
    toolset="my_toolset",
    schema={
        "type": "function",
        "function": {
            "name": "my_action",
            "description": "执行我的自定义操作",
            "parameters": {
                "type": "object",
                "properties": {
                    "param1": {
                        "type": "string",
                        "description": "参数1的说明"
                    }
                },
                "required": ["param1"]
            }
        }
    },
    handler=handle_my_action,
    check_fn=check_my_tool_available,
    requires_env=["MY_API_KEY"],
    is_async=True,
    description="我的自定义工具",
    emoji="🔧",
)
```

### 步骤 2：注册到 discover 流程

在 `model_tools.py` 的 `_discover_tools()` 里加入你的工具模块。

### 步骤 3：必要时加 toolset

如果你定义了新的 `toolset`，在 `toolsets.py` 里补上。

### 步骤 4：写测试

```python
import pytest
from tools.my_tool import handle_my_action


@pytest.mark.asyncio
async def test_my_action():
    result = await handle_my_action({"param1": "test"})
    assert "处理了" in result
```

### 注意事项

- handler 必须返回**字符串**，实际项目里通常返回 JSON 字符串
- 不要让工具冒出未捕获异常
- `check_fn` 决定的是“工具是否可见”，不是“调用后是否能成功”

---

## 添加新的消息平台

### 步骤 1：创建平台适配器

```python
from gateway.platforms.base import BasePlatform


class MyPlatformAdapter(BasePlatform):
    async def start(self):
        pass

    async def stop(self):
        pass

    async def send_message(self, chat_id: str, text: str, **kwargs):
        pass
```

### 步骤 2：注册平台

在 `gateway/config.py` 的 `Platform` 枚举中加入新平台。

### 步骤 3：编写测试

在 `tests/gateway/` 下补测试。

---

## 添加新的记忆提供商

如果你要做长期记忆后端，不要走普通工具扩展，应该看：

- [plugin-integration-guide.md](./plugin-integration-guide.md)
- `agent/memory_provider.py`
- `agent/memory_manager.py`
- `plugins/memory/`

---

## 一次典型开发闭环

假设你要改一个工具，推荐这个顺序：

1. 读实现文件 `tools/<tool>.py`
2. 读相关测试
3. 小步修改
4. 跑最小测试
5. 用 CLI 做一次 smoke test
6. 再决定要不要跑更大范围测试

这样比“先改一堆，再一次性跑全量测试”更稳。

---

## 调试技巧

### 启用详细日志

```bash
HERMES_LOG_LEVEL=DEBUG hermes
```

### 查看 API 请求

```bash
HERMES_LOG_API=1 hermes
```

### 检查工具注册状态

```python
from tools.registry import registry
print(registry.get_tool_to_toolset_map())
```

### 直接调工具

```python
from model_tools import handle_function_call
result = handle_function_call("web_search", {"query": "test"})
print(result)
```

---

## 常见坑

### 1. 忘记激活 venv

一切奇怪的依赖问题都先排查这个。

### 2. 文档里的名字和代码当前名字不一致

优先相信：

- `hermes_cli/commands.py`
- `toolsets.py`
- `model_tools.py`
- `tools/registry.py`

### 3. 把 Skills、Tools、Plugins 混为一谈

它们不是一回事：

- Tool：Python 能力
- Skill：Markdown 指南
- Plugin：运行时扩展机制

### 4. 一上来就跑完整测试矩阵

第一次做小改动时，先跑最小相关测试。

---

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `openai` | OpenAI 兼容 API 客户端 |
| `anthropic` | Anthropic 专用路径和能力 |
| `httpx` | 异步 HTTP |
| `prompt_toolkit` | CLI 输入体验 |
| `rich` | 终端展示 |
| `pydantic` | 数据验证 |
| `tenacity` | 重试逻辑 |

---

## 代码约定

- Python 3.11+
- 工具 handler 返回字符串，不直接抛未捕获异常
- 每个工具一个文件，自注册
- 改动尽量伴随最小测试
- 路径和持久化状态优先走 `HERMES_HOME` 作用域
