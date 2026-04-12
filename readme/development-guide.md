# 开发者指南

本文档面向想要为 Hermes Agent 贡献代码、添加功能或进行二次开发的开发者。

---

## 开发环境搭建

### 1. 克隆并安装

```bash
git clone <repo-url> hermes-agent
cd hermes-agent

# 使用 uv（推荐）
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"

# 可选：RL 训练子模块
git submodule update --init tinker-atropos
uv pip install -e "./tinker-atropos"

# 可选：浏览器工具
npm install
```

### 2. 验证安装

```bash
hermes doctor
pytest tests/ -v --co  # 列出所有测试（不执行）
```

---

## 运行测试

```bash
# 运行全部测试
pytest tests/ -v

# 运行单个文件
pytest tests/test_agent_loop.py -v

# 运行单个测试
pytest tests/test_agent_loop.py::test_name -v

# 运行特定目录
pytest tests/gateway/ -v
pytest tests/hermes_cli/ -v
pytest tests/agent/ -v

# 带覆盖率
pytest tests/ --cov=. --cov-report=html
```

### 测试目录结构

```
tests/
├── test_agent_loop*.py      # 核心循环测试
├── test_anthropic_*.py      # Anthropic API 测试
├── test_cli_*.py            # CLI 测试
├── test_model_*.py          # 模型相关测试
├── test_compression_*.py    # 压缩测试
├── acp/                     # ACP 服务器测试
├── agent/                   # agent/ 模块测试
├── cron/                    # 定时任务测试
├── gateway/                 # 网关平台测试
├── hermes_cli/              # CLI 命令测试
├── integration/             # 集成测试
├── plugins/                 # 插件测试
├── skills/                  # 技能测试
└── e2e/                     # 端到端测试
```

---

## 代码结构速查

当你想修改某个功能时，这张表帮你快速定位到对应文件：

| 我想修改... | 去找... |
|------------|---------|
| 智能体的核心对话循环 | `run_agent.py` → `AIAgent` 类 |
| 系统提示词的组装方式 | `agent/prompt_builder.py` |
| 上下文压缩的算法 | `agent/context_compressor.py` |
| 模型元数据（context length 等） | `agent/model_metadata.py` |
| 记忆系统的行为 | `agent/memory_manager.py` |
| 某个工具的实现 | `tools/<tool_name>.py` |
| 工具的注册机制 | `tools/registry.py` |
| 工具集的分组定义 | `toolsets.py`、`toolset_distributions.py` |
| 工具发现和分发逻辑 | `model_tools.py` |
| CLI 的交互界面 | `cli.py` |
| CLI 子命令 | `hermes_cli/main.py`、`hermes_cli/commands.py` |
| 某个平台的网关适配 | `gateway/platforms/<platform>.py` |
| 网关的会话管理 | `gateway/session.py` |
| 定时任务系统 | `cron/scheduler.py`、`cron/jobs.py` |
| 技能的解析逻辑 | `agent/skill_utils.py` |
| 编辑器集成（ACP） | `acp_adapter/` |
| 安装脚本 | `scripts/install.sh`、`scripts/install.ps1` |
| Docker 配置 | `Dockerfile`、`docker/` |

---

## 添加新工具

这是最常见的扩展场景。按以下步骤操作：

### 步骤 1：创建工具文件

在 `tools/` 目录下创建新文件，如 `tools/my_tool.py`：

```python
"""我的自定义工具"""

from tools.registry import registry


def check_my_tool_available():
    """检查工具是否可用（环境变量、依赖等）"""
    import os
    return os.environ.get("MY_API_KEY") is not None


async def handle_my_action(arguments: dict, **kwargs) -> str:
    """工具的实际执行逻辑"""
    param1 = arguments.get("param1", "")

    # 你的业务逻辑
    result = f"处理了: {param1}"

    return result


# 自注册（模块导入时执行）
registry.register(
    name="my_action",
    toolset="my_toolset",        # 工具集名称
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

### 步骤 2：注册到工具发现系统

在 `model_tools.py` 中添加导入：

```python
# 在 _discover_tools() 函数中添加
from tools import my_tool
```

### 步骤 3：添加到工具集定义（可选）

如果创建了新的工具集，在 `toolsets.py` 中注册：

```python
TOOLSETS["my_toolset"] = ["my_action"]
```

### 步骤 4：编写测试

在 `tests/` 下创建测试文件：

```python
# tests/test_my_tool.py
import pytest
from tools.my_tool import handle_my_action


@pytest.mark.asyncio
async def test_my_action():
    result = await handle_my_action({"param1": "test"})
    assert "处理了" in result
```

---

## 添加新的消息平台

### 步骤 1：创建平台适配器

在 `gateway/platforms/` 下创建新文件：

```python
# gateway/platforms/my_platform.py

from gateway.platforms.base import BasePlatform


class MyPlatformAdapter(BasePlatform):
    """我的平台适配器"""

    async def start(self):
        """启动平台连接"""
        pass

    async def stop(self):
        """停止平台连接"""
        pass

    async def send_message(self, chat_id: str, text: str, **kwargs):
        """发送消息到平台"""
        pass

    async def on_message(self, message):
        """处理收到的消息"""
        # 转换为统一格式后交给 GatewayRunner
        pass
```

### 步骤 2：注册平台

在 `gateway/config.py` 的 `Platform` 枚举中添加：

```python
class Platform(str, Enum):
    MY_PLATFORM = "my_platform"
    # ...
```

### 步骤 3：编写测试

在 `tests/gateway/` 下创建测试。

---

## 添加新的记忆提供商

### 步骤 1：创建插件目录

```bash
mkdir -p plugins/memory/my_memory/
```

### 步骤 2：实现插件

```python
# plugins/memory/my_memory/__init__.py

def register(ctx):
    """注册记忆提供商"""
    from agent.memory_provider import MemoryProvider

    class MyMemoryProvider(MemoryProvider):
        async def prefetch(self, user_message: str) -> str:
            """预取相关记忆"""
            pass

        async def sync(self, user_msg: str, assistant_response: str):
            """同步记忆"""
            pass

    ctx.add_provider(MyMemoryProvider())
```

---

## 代码风格与约定

### 通用约定

- Python 3.11+ 类型注解
- 使用 `async/await` 处理 I/O 密集操作
- 使用 `pydantic` 做数据验证
- 使用 `rich` 做终端输出格式化
- 使用 `tenacity` 做重试逻辑

### 文件组织

- 每个工具一个文件，自注册
- 每个平台适配器一个文件
- 测试文件与源文件对应
- 使用相对导入（包内）

### 错误处理

- 工具 handler 返回字符串（成功和失败都是）
- 不要让工具抛出未捕获异常
- 使用 `hermes_logging` 记录日志
- 原子写入保护配置文件

---

## 关键依赖说明

| 依赖 | 用途 | 注意事项 |
|------|------|----------|
| `openai` | LLM API 调用 | 所有提供商都走 OpenAI 兼容接口 |
| `anthropic` | Anthropic 专用功能 | Prompt caching 等 |
| `httpx` | 异步 HTTP | 替代 requests |
| `prompt_toolkit` | TUI 框架 | 多行编辑、自动补全 |
| `rich` | 终端美化 | 表格、颜色、进度条 |
| `pydantic` | 数据验证 | 配置和 Schema |
| `tenacity` | 重试逻辑 | API 调用的指数退避 |
| `jinja2` | 模板引擎 | 提示词模板 |

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

### 查看工具调用

在聊天中使用 `/usage` 查看 token 消耗和工具调用记录。

### 检查工具注册状态

```python
from tools.registry import registry
print(registry.get_tool_to_toolset_map())
```

### 运行单个工具（调试用）

```python
from model_tools import handle_function_call
result = handle_function_call("web_search", {"query": "test"})
print(result)
```

---

## RL 训练相关

项目包含 RL 训练基础设施：

| 文件/目录 | 说明 |
|-----------|------|
| `batch_runner.py` | 批量轨迹生成 |
| `trajectory_compressor.py` | 轨迹压缩（训练数据预处理） |
| `rl_cli.py` | RL 训练 CLI |
| `environments/` | RL 训练环境（基准测试） |
| `agent/trajectory.py` | 轨迹格式定义 |
| `tinker-atropos/` | RL 训练子模块（git submodule） |

---

## 发布流程

```bash
# 使用发布脚本
python scripts/release.py

# Docker 构建
docker build -t hermes-agent .
```

---

## 常见问题

**Q: 为什么 `run_agent.py` 这么大（9000+ 行）？**
A: 这是核心编排器，包含了对话循环的所有逻辑。部分模块已经拆分到 `agent/` 目录下，但核心循环仍然在这个文件中。

**Q: 异步工具怎么在同步上下文中调用？**
A: `model_tools.py` 中有 `_run_async()` 函数，通过持久化的事件循环做异步桥接。主线程和工作线程各有独立的事件循环。

**Q: 如何测试需要 API 密钥的工具？**
A: 使用 mock。大部分测试用 `unittest.mock` 模拟 API 响应。参见 `tests/` 中的现有测试。

**Q: 技能和工具有什么区别？**
A: 工具是硬编码的 Python 函数（有 JSON Schema），技能是动态的 Markdown 文档（增强提示词）。工具是"能力"，技能是"知识"。
