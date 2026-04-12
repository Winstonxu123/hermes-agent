# CLI 命令指南

Hermes Agent 提供一个功能丰富的交互式终端界面（TUI），基于 prompt_toolkit 构建，支持多行输入、自动补全、语法高亮等。

---

## 启动方式

```bash
# 启动交互式终端（默认命令）
hermes

# 等同于
hermes chat

# 运行一次性查询
hermes chat -q "帮我搜索最新的 AI 新闻"

# 指定模型
hermes chat --model claude-3-opus

# 指定个性
hermes chat --personality researcher
```

---

## 主要子命令

| 命令 | 说明 |
|------|------|
| `hermes` / `hermes chat` | 启动交互式对话 |
| `hermes setup` | 运行交互式配置向导 |
| `hermes model` | 切换 LLM 模型 |
| `hermes tools` | 管理工具启用状态 |
| `hermes gateway` | 管理消息网关（启动/停止/状态） |
| `hermes cron` | 管理定时任务 |
| `hermes skills` | 浏览和管理技能 |
| `hermes doctor` | 诊断安装问题 |
| `hermes status` | 查看组件状态 |
| `hermes acp` | 启动编辑器集成服务（ACP 协议） |

---

## 聊天中的斜杠命令

在交互式对话中，可以使用以下斜杠命令：

### 会话管理

| 命令 | 说明 |
|------|------|
| `/new` | 开始新会话 |
| `/clear` | 清空当前会话 |
| `/history` | 浏览历史会话 |
| `/save` | 保存当前会话 |
| `/retry` | 重新生成最后一条回复 |
| `/undo` | 撤销最后一轮对话 |
| `/branch` | 从当前点创建会话分支 |
| `/compress` | 手动触发上下文压缩 |

### 配置切换

| 命令 | 说明 |
|------|------|
| `/model` | 切换模型 |
| `/provider` | 切换 API 提供商 |
| `/prompt` | 修改系统提示词 |
| `/personality` | 切换人格/个性 |
| `/voice` | 启用/禁用语音模式 |
| `/reasoning` | 调整推理深度（effort level） |

### 工具与技能

| 命令 | 说明 |
|------|------|
| `/tools` | 查看/切换工具状态 |
| `/toolsets` | 管理工具集 |
| `/skills` | 浏览技能 |
| `/browser` | 浏览器工具设置 |
| `/cron` | 管理定时任务 |

### 信息查看

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息 |
| `/usage` | 查看 token 使用量 |
| `/insights` | 查看使用洞察 |
| `/status` | 查看状态仪表板 |
| `/platforms` | 查看已连接平台 |

### 退出

| 命令 | 说明 |
|------|------|
| `/quit` | 退出 |
| `/exit` | 退出 |

---

## 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 中断当前操作（可输入新指令重定向） |
| `Ctrl+D` | 退出 |
| `Tab` | 自动补全斜杠命令 |
| `↑` / `↓` | 浏览输入历史 |

---

## 状态栏

终端底部显示状态栏，包含：

- 当前模型名称
- Token 使用量 / 上下文窗口大小
- 当前工具集状态
- 流式输出状态

---

## 配置文件

CLI 的行为主要由 `~/.hermes/config.yaml` 控制。详见 [配置参考](./configuration-reference.md)。

最常用的配置项：

```yaml
# 默认模型
model:
  default: "anthropic/claude-sonnet-4-20250514"

# 显示设置
display:
  compact: false       # 紧凑模式
  streaming: true      # 流式输出
  show_reasoning: true # 显示推理过程
  skin: default        # 主题皮肤

# 智能体行为
agent:
  max_turns: 50        # 单次对话最大轮数
```

---

## 多配置文件（Profiles）

支持维护多个独立的配置环境：

```bash
# 使用名为 "work" 的配置文件
hermes --profile work

# 使用名为 "research" 的配置文件
hermes --profile research
```

每个 Profile 有独立的 `config.yaml`、`.env`、技能、记忆等。
