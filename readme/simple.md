# Hermes Agent 新人 Onboarding

这份文档是给**第一次进入 Hermes 仓库的开发者**准备的。目标不是一次看懂全部实现，而是让你在前 30 分钟内完成下面 4 件事：

1. 知道项目是干什么的
2. 知道第一天应该先看哪些文件
3. 知道怎么把项目跑起来
4. 知道第一个改动应该从哪里下手

更详细的材料请继续阅读：

- [开发者指南](./development-guide.md)
- [系统架构](./architecture.md)
- [CLI 命令指南](./cli-guide.md)
- [配置参考](./configuration-reference.md)

---

## 30 分钟上手路径

### 第 1 步：把环境跑通

```bash
git clone <repo-url> hermes-agent
cd hermes-agent

uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"

hermes doctor
hermes
```

仓库里的硬约定是：

```bash
source venv/bin/activate
```

只要你准备跑 Python 命令、测试或 CLI，先做这一步。

### 第 2 步：先建立一个粗模型

Hermes 可以先粗暴地理解成：

```text
用户输入
-> AIAgent 组装 prompt 和工具列表
-> 调用 LLM
-> 如果返回 tool_calls 就执行工具
-> 把工具结果塞回去
-> 再调 LLM
-> 直到拿到最终回答
```

目录多，不代表你第一天要全看。先抓住 3 条主线：

1. `run_agent.py`：主循环，所有能力最终汇总到这里。
2. `model_tools.py` + `tools/registry.py`：工具是怎么被发现、注册、暴露给模型的。
3. `cli.py` / `gateway/run.py`：同一个 agent 怎么被终端或消息平台驱动。

### 第 3 步：按目标找文件

| 你的目标 | 先看这些文件 |
|---------|--------------|
| 理解整体架构 | `readme/architecture.md`、`run_agent.py` |
| 改一个工具 | `tools/<tool>.py`、`tools/registry.py`、`model_tools.py` |
| 改系统提示词 | `agent/prompt_builder.py` |
| 改上下文压缩 | `agent/context_compressor.py` |
| 改 CLI 行为 | `cli.py`、`hermes_cli/main.py`、`hermes_cli/commands.py` |
| 改消息平台 | `gateway/run.py`、`gateway/platforms/<platform>.py` |
| 加技能 | `skills/`、`agent/skill_utils.py`、`tools/skills_tool.py` |
| 加 plugin/provider | `plugins/`、`hermes_cli/plugins.py` |

### 第 4 步：选一个最小改动

第一次提改动，优先选这些：

1. 修文档和代码不一致
2. 给现有工具补测试
3. 修一个 CLI 文案或帮助信息
4. 修一个配置项说明
5. 修一个小型工具暴露或展示问题

不建议第一次就做：

- 大规模拆 `run_agent.py`
- 同时动 CLI、gateway、tool registry 三层
- 一口气重构工具和提示词系统

---

## 第一天推荐阅读顺序

### 第一天

1. 读完这份文档
2. 跑一次 `hermes`
3. 读 [development-guide.md](./development-guide.md)
4. 读 [architecture.md](./architecture.md) 的前半部分

### 第二天

1. 看 `run_agent.py` 里的 `AIAgent.__init__`
2. 看 `run_conversation()`
3. 配合 [run_agent-reading-map.md](./run_agent-reading-map.md) 跳读

### 第三天以后

按目标继续：

- 工具：看 [tools-reference.md](./tools-reference.md)
- Skills：看 [skills-guide.md](./skills-guide.md)
- Gateway：看 [gateway-guide.md](./gateway-guide.md)
- Plugin：看 [plugin-integration-guide.md](./plugin-integration-guide.md)

---

## 你应该分清楚的 4 个概念

| 概念 | 本质 | 位置 |
|------|------|------|
| `AIAgent` | 主循环编排器 | `run_agent.py` |
| Tool | 模型可调用的 Python 能力 | `tools/` |
| Skill | Markdown 操作指南 / SOP | `skills/` |
| Plugin | 运行时扩展机制 | `plugins/`、`~/.hermes/plugins/` |

一句话记忆：

- Tool 决定“能做什么”
- Skill 决定“应该怎么做”
- Plugin 决定“怎么接新的运行时扩展”

---

## 新人最容易踩的坑

### 1. 忘记激活虚拟环境

很多奇怪的 `ImportError`、命令找不到、测试不一致，最后都是这个原因。

### 2. 一上来就从 `run_agent.py` 第 1 行读到最后

这会很痛苦。更高效的顺序是：

1. `simple.md`
2. `architecture.md`
3. `run_agent-reading-map.md`
4. 再按功能跳读 `run_agent.py`

### 3. 以为文档里的旧名字一定是当前实现

有些名称是历史遗留。碰到不一致时，优先相信：

- `hermes_cli/commands.py`
- `toolsets.py`
- `model_tools.py`
- `tools/registry.py`

### 4. 一上来就跑全量测试

先跑最小相关测试：

```bash
python -m pytest tests/hermes_cli/ -q
python -m pytest tests/agent/ -q
```

确认方向正确后，再扩大测试范围。

---

## 新人调试套路

出现问题时，按这个顺序排查通常比较稳：

1. `source venv/bin/activate`
2. `hermes doctor`
3. `HERMES_LOG_LEVEL=DEBUG hermes`
4. 看配置：`~/.hermes/config.yaml`
5. 看工具注册：`tools/registry.py` / `model_tools.py`
6. 跑最小测试复现

如果怀疑工具没有正确暴露，可以直接打印：

```python
from tools.registry import registry
print(registry.get_tool_to_toolset_map())
```

---

## 作为开发者，你可以把 Hermes 理解成什么

如果你熟悉别的 Agent 框架，可以这样类比：

- 像 ChatGPT Agent：但本地控制感更强
- 像 LangChain/AutoGen：但主循环更集中在 `AIAgent`
- 像一个终端 Copilot：CLI 是一等入口
- 像一个多平台 Bot：gateway 能把它挂到 Telegram / Discord / Slack

最重要的不是第一天就“全部理解”，而是先形成稳定的心理模型：

> Hermes = 一个带工具系统、技能系统、记忆系统和多入口运行时的 Agent 编排器。
