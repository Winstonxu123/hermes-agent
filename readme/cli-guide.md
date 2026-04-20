# CLI 命令指南

Hermes 的 CLI 是新人最应该先熟悉的入口。因为绝大多数问题在 CLI 模式下都更容易复现、观察和调试。

如果你还没读过 [simple.md](./simple.md)，建议先读那篇再回来。

---

## 5 分钟上手

```bash
source venv/bin/activate
hermes doctor
hermes
```

进入 CLI 后，先试这几个命令：

```text
/help
/status
/model
/tools
/usage
```

如果你只想知道“还有哪些命令”，直接输入：

```text
/commands
```

---

## 启动方式

```bash
# 启动交互式终端
hermes

# 等同于
hermes chat

# 一次性执行
hermes chat -q "帮我总结这个仓库的结构"

# 指定模型
hermes chat --model anthropic/claude-sonnet-4-20250514

# 指定人格
hermes chat --personality researcher
```

对新人最常用的通常只有两种：

- `hermes`
- `hermes chat -q "..."`

---

## 常见子命令

| 命令 | 说明 |
|------|------|
| `hermes` / `hermes chat` | 启动交互式对话或一次性对话 |
| `hermes setup` | 运行交互式配置向导 |
| `hermes model` | 切换或查看模型 |
| `hermes tools` | 管理工具启用状态 |
| `hermes skills` | 浏览和管理技能 |
| `hermes gateway` | 管理消息网关 |
| `hermes cron` | 管理定时任务 |
| `hermes doctor` | 诊断环境和依赖 |
| `hermes status` | 查看运行状态 |
| `hermes acp` | 启动编辑器集成服务 |

---

## 聊天中的斜杠命令

下面按“会话管理 / 配置 / 工具 / 信息查看”分组列出最常用的命令。完整列表以 `/commands` 和 `hermes_cli/commands.py` 为准。

### 会话管理

| 命令 | 说明 |
|------|------|
| `/new` | 开始新会话 |
| `/clear` | 清空当前会话 |
| `/history` | 浏览历史会话 |
| `/save` | 保存当前会话 |
| `/retry` | 重试上一条回答 |
| `/undo` | 撤销最后一轮 |
| `/title [name]` | 设置当前会话标题 |
| `/branch [name]` | 从当前节点分叉 |
| `/compress [focus topic]` | 手动触发上下文压缩 |
| `/rollback [number]` | 回滚到较早节点 |
| `/snapshot [create|restore <id>|prune]` | 管理会话快照 |
| `/restart` | 重启当前 CLI 会话 |
| `/stop` | 停止当前执行中的 agent |

### 配置切换

| 命令 | 说明 |
|------|------|
| `/model` | 切换模型 |
| `/provider` | 切换提供商 |
| `/personality` | 切换人格 |
| `/voice [on|off|tts|status]` | 控制语音模式 |
| `/reasoning [level|show|hide]` | 调整推理深度或显示方式 |
| `/config` | 查看配置入口 |
| `/statusbar` | 切换状态栏 |
| `/verbose` | 切换详细输出 |
| `/fast [normal|fast|status]` | 切换快速模式 |
| `/skin [name]` | 切换 CLI 皮肤 |

### 工具与技能

| 命令 | 说明 |
|------|------|
| `/tools` | 查看和切换工具状态 |
| `/toolsets` | 管理 toolset |
| `/skills` | 浏览技能 |
| `/browser [connect|disconnect|status]` | 管理浏览器能力 |
| `/cron [subcommand]` | 管理定时任务 |
| `/plugins` | 查看插件状态 |
| `/reload` | 重载部分运行时状态 |
| `/reload-mcp` | 重新发现 MCP 工具 |

### 信息查看

| 命令 | 说明 |
|------|------|
| `/help` | 查看帮助 |
| `/commands [page]` | 分页查看所有命令 |
| `/status` | 查看当前状态 |
| `/usage` | 查看 token 使用量 |
| `/insights [days]` | 查看使用洞察 |
| `/platforms` | 查看已连接平台 |
| `/profile` | 查看当前 profile |
| `/paste` | 处理大段粘贴内容 |
| `/image <path>` | 发送本地图像到会话 |
| `/update` | 查看更新信息 |
| `/debug` | 查看调试信息 |

### 后台与审批

| 命令 | 说明 |
|------|------|
| `/background <prompt>` | 后台执行任务 |
| `/queue <prompt>` | 把任务排队到下一条 |
| `/btw <question>` | 临时插入一个补充问题 |
| `/approve [session|always]` | 调整危险操作审批 |
| `/deny` | 拒绝当前审批请求 |

### 退出

| 命令 | 说明 |
|------|------|
| `/quit` | 退出 |
| `/exit` | 退出 |

---

## 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 中断当前操作 |
| `Ctrl+D` | 退出 CLI |
| `Tab` | 自动补全命令或参数 |
| `↑` / `↓` | 浏览输入历史 |

补充说明：

- `Ctrl+C` 在 Hermes 里通常更像“中断并允许你改派任务”，不是简单粗暴的直接退出。
- 复杂提示词和长段文本可以直接粘贴到输入框。

---

## 状态栏里能看到什么

终端底部状态栏通常会显示：

- 当前模型
- 当前 provider
- token / context 使用情况
- 部分 toolset 或运行态信息

如果你在调试模型选择、上下文压力或工具状态，先看状态栏通常很有帮助。

---

## 最常用的配置项

CLI 行为主要由 `~/.hermes/config.yaml` 控制，详见 [configuration-reference.md](./configuration-reference.md)。

新人通常先关心这些：

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: "openrouter"

display:
  skin: "default"
  streaming: true
  show_reasoning: true

agent:
  max_turns: 50
```

---

## Profiles

Hermes 支持多 profile，方便把不同用途隔离开：

```bash
hermes --profile work
hermes --profile personal
hermes --profile safe
```

每个 profile 都有自己独立的：

- `config.yaml`
- `.env`
- skills
- 记忆
- session 数据

推荐给新人的最简单用法：

- `default`：个人日常
- `work`：工作环境
- `safe`：最小权限实验环境

---

## 推荐的日常工作流

### 调试一个问题

1. `source venv/bin/activate`
2. `hermes`
3. `/status`
4. `/tools`
5. `/usage`

### 比较两个模型的行为

1. `/model`
2. `/reasoning`
3. `/retry`

### 看某个工具是否真的暴露了

1. `/tools`
2. `/toolsets`
3. 还不确定的话，退出后跑 `hermes tools list --platform cli`

### 调试 gateway 问题

1. 先在 CLI 复现
2. 再去看 `hermes gateway status`
3. 最后读 [gateway-guide.md](./gateway-guide.md)
