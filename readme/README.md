# Hermes Agent 文档中心

> Hermes Agent 是由 Nous Research 开发的自我进化 AI 智能体，能够从经验中创建技能、在使用中改进，并可运行在多种平台上。

## 文档导航

| 文档 | 说明 | 适合谁读 |
|------|------|----------|
| [快速上手指南](#快速上手指南) | 安装、配置、第一次运行 | 所有人 |
| [系统架构](./architecture.md) | 核心循环、模块关系、设计模式 | 想理解代码的开发者 |
| [CLI 命令指南](./cli-guide.md) | 交互式终端的所有命令和用法 | 日常使用者 |
| [工具参考手册](./tools-reference.md) | 50+ 工具的完整列表和说明 | 开发者 / 高级用户 |
| [Skills 技能系统](./skills-guide.md) | 技能的使用、创建、发布 | 想扩展功能的用户 |
| [消息网关指南](./gateway-guide.md) | Telegram/Discord/Slack 等平台接入 | 想部署到聊天平台的用户 |
| [配置参考](./configuration-reference.md) | 所有配置项的完整说明 | 需要自定义行为的用户 |
| [开发者指南](./development-guide.md) | 如何贡献代码、添加工具、写测试 | 贡献者 / 二次开发者 |

---

## 快速上手指南

### 1. 环境要求

- Python 3.11+
- Node.js（可选，用于浏览器工具）
- Git

### 2. 安装

**方式一：使用 uv（推荐）**

```bash
# 安装 uv 包管理器
curl -LsSf https://astral.sh/uv/install.sh | sh

# 克隆项目
git clone <repo-url> hermes-agent
cd hermes-agent

# 创建虚拟环境并安装依赖
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
```

**方式二：使用 pip**

```bash
python -m venv venv
source venv/bin/activate
pip install -e ".[all,dev]"
```

**方式三：使用安装脚本**

```bash
# Linux / macOS
bash scripts/install.sh

# Windows (PowerShell)
powershell scripts/install.ps1
```

**方式四：使用 Docker**

```bash
docker build -t hermes-agent .
docker run -it --env-file .env hermes-agent
```

### 3. 配置 API 密钥

复制示例配置并填入你的 API 密钥：

```bash
cp .env.example ~/.hermes/.env
```

编辑 `~/.hermes/.env`，至少配置一个 LLM 提供商：

```env
# 任选一个（或多个）
OPENROUTER_API_KEY=sk-or-...        # OpenRouter（推荐，支持 100+ 模型）
ANTHROPIC_API_KEY=sk-ant-...         # Anthropic Claude
OPENAI_API_KEY=sk-...                # OpenAI GPT
NOUS_API_KEY=...                     # Nous Research Portal
```

### 4. 验证安装

```bash
hermes doctor
```

此命令会检查：Python 版本、依赖安装、API 密钥配置、工具可用性等。

### 5. 第一次运行

```bash
# 启动交互式终端
hermes

# 或运行一次性查询
hermes chat -q "你好，请介绍一下你自己"
```

### 6. 交互式设置向导

如果你不确定如何配置，可以运行：

```bash
hermes setup
```

这会引导你完成模型选择、工具启用、消息平台配置等所有步骤。

---

## 项目是什么？能做什么？

Hermes Agent 是一个**通用型 AI 智能体**，它不仅仅是一个聊天机器人，而是一个能够：

- **执行终端命令** — 在本地、Docker、SSH、云端运行代码
- **浏览网页** — 搜索信息、提取网页内容、自动化浏览器操作
- **读写文件** — 创建、编辑、管理本地文件
- **管理定时任务** — 设置定期执行的自动化工作
- **跨平台通信** — 通过 Telegram、Discord、Slack、WhatsApp 等平台交互
- **自我学习** — 将复杂任务提炼为可复用的"技能"（Skills）
- **持久记忆** — 记住用户偏好和重要上下文
- **多模型支持** — 兼容 OpenAI、Anthropic、OpenRouter 等 100+ 模型

---

## 项目目录结构概览

```
hermes-agent/
├── run_agent.py          # 核心智能体循环（入口之一）
├── cli.py                # 交互式终端 UI
├── model_tools.py        # 工具发现与分发
├── toolsets.py           # 工具集分组管理
├── hermes_constants.py   # 全局常量
├── agent/                # 智能体内部模块（提示词构建、压缩、模型元数据等）
├── tools/                # 50+ 自注册工具实现
├── gateway/              # 消息平台网关（Telegram、Discord 等）
├── hermes_cli/           # CLI 命令实现
├── skills/               # 内置技能（30+ 类别）
├── optional-skills/      # 可选技能（通过 hub 安装）
├── plugins/              # 插件系统（记忆提供商等）
├── acp_adapter/          # 编辑器集成（VS Code、Cursor）
├── environments/         # RL 训练环境
├── cron/                 # 定时任务调度
├── tests/                # 测试套件（150+ 测试文件）
├── scripts/              # 安装和发布脚本
├── doc/                  # 📖 你正在阅读的文档
└── pyproject.toml        # 项目元数据和依赖
```

> 想深入了解各模块的作用和关系？请阅读 [系统架构](./architecture.md)。
