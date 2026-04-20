# 配置参考

Hermes 的配置主要分成两类：

- `~/.hermes/.env`：放敏感信息和外部服务凭证
- `~/.hermes/config.yaml`：放行为、模型、工具、gateway、skills 等配置

如果你是新人，不用一开始就把整篇背下来。先理解：

1. 配置优先级
2. 最小可用配置
3. 哪些字段最常改

---

## 配置优先级

排查“为什么我改了配置没生效”时，优先按这个顺序看：

1. 命令行参数
2. 环境变量
3. 当前 profile 下的 `config.yaml` / `.env`
4. 代码默认值

典型例子：

- 你明明改了 `config.yaml` 的模型，但 `HERMES_MODEL` 环境变量把它覆盖了
- 你以为自己改的是默认配置，实际上当前跑的是 `--profile work`

---

## 最小可用配置

大多数新人只需要一个模型密钥就能把 CLI 跑起来：

```env
# ~/.hermes/.env
OPENROUTER_API_KEY=sk-or-...
```

```yaml
# ~/.hermes/config.yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4-20250514

display:
  skin: default

agent:
  max_turns: 50
```

在这基础上，再逐步添加：

- `toolsets`
- `skills`
- `gateway`
- `memory`

---

## 新人最容易搞混的 3 件事

### 1. `.env` 和 `config.yaml` 不是一回事

- `.env` 放 token、API key、secret
- `config.yaml` 放行为开关和结构化配置

经验法则：

- 能泄露权限的值，优先放 `.env`
- 会被团队讨论、希望可读可审查的行为配置，放 `config.yaml`

### 2. 当前生效的是哪个 profile

如果你用了：

```bash
hermes --profile work
```

那你改的就不再是默认的 `~/.hermes/config.yaml`，而是对应 profile 下的配置。

排查时先确认自己到底跑的是：

- `default`
- `work`
- `personal`
- 还是别的 profile

### 3. 配置值不生效，不一定是 YAML 写错

更常见的原因是：

- 被环境变量覆盖了
- 当前 profile 不是你以为的那个
- 入口不同，CLI 和 gateway 读取的配置路径或默认值不完全一样

---

## 目录结构

```text
~/.hermes/
├── config.yaml          # 主配置文件
├── .env                 # API key / token / 敏感变量
├── skills/              # 活跃技能
├── memories/            # 内置持久化记忆
│   ├── MEMORY.md
│   └── USER.md
├── state.db             # SQLite session 数据
├── sessions/            # 会话日志
├── cron/                # 定时任务数据
├── cache/               # 元数据和缓存
└── profiles/            # 多 profile
```

---

## `.env`

敏感信息优先放 `.env`，而不是硬编码在 `config.yaml`。

```env
# ===== LLM 提供商 =====
OPENROUTER_API_KEY=sk-or-...
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
NOUS_API_KEY=...

# ===== Web / Browser / Media 工具 =====
EXA_API_KEY=...
FIRECRAWL_API_KEY=...
FAL_KEY=...
ELEVENLABS_API_KEY=...

# ===== 消息平台 =====
TELEGRAM_BOT_TOKEN=...
DISCORD_BOT_TOKEN=...
SLACK_BOT_TOKEN=...
SLACK_APP_TOKEN=...

# ===== 其他集成 =====
HOMEASSISTANT_TOKEN=...
GITHUB_TOKEN=...
```

经验法则：

- 会泄露权限的值都放 `.env`
- 团队共享仓库不要提交个人 `.env`
- 调 profile 问题时，记得看的是对应 profile 的 `.env`

---

## `config.yaml`

### `model`

最常改的一组配置。

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: "openrouter"
  base_url: null
```

建议：

- 新人先只改 `default` 和 `provider`
- 只有在接自定义兼容端点时才碰 `base_url`

### `terminal`

决定终端工具跑在哪。

```yaml
terminal:
  env_type: "local"      # local/docker/ssh/modal/daytona/singularity
  cwd: null
  timeout: 120

  docker_image: "ubuntu:22.04"
  docker_volumes: []

  ssh_host: null
  ssh_user: null
  ssh_key: null
```

新人建议：

- 先用 `local`
- 真的需要隔离环境时再切 Docker 或 SSH

### `browser`

```yaml
browser:
  inactivity_timeout: 300
  record_sessions: false
  provider: "browser_use"
```

### `compression`

```yaml
compression:
  enabled: true
  threshold: 0.5
  summary_model: null
```

如果你在调“上下文为什么突然被压缩”，重点看这里和 `agent/context_compressor.py`。

### `agent`

```yaml
agent:
  max_turns: 50
  system_prompt: null
  personalities: {}
```

这里最常用的是：

- `max_turns`
- `system_prompt`

### `display`

```yaml
display:
  compact: false
  skin: "default"
  streaming: true
  show_reasoning: true
```

### `delegation`

```yaml
delegation:
  max_iterations: 20
  default_toolsets:
    - terminal
    - file
    - web
  model: null
  base_url: null
```

### `toolsets`

```yaml
toolsets:
  enabled:
    - web
    - terminal
    - file
    - skills
    - memory
    - cronjob
    - delegation
  disabled:
    - browser
    - vision
```

注意：

- toolset 的真实名字以 `toolsets.py` 为准
- 如果你看到旧文档里提到 `cron`、`delegate` 一类的历史名称，以当前实现为准

### `skills`

```yaml
skills:
  disabled:
    - social-media/xitter
  external_dirs:
    - /path/to/custom/skills
```

### `memory`

```yaml
memory:
  provider: ""            # 空字符串表示只用内置记忆；也可以写具体 provider 名
  nudge_interval: 10
```

如果你没接外部 memory provider，很多时候这里保持默认即可。

### `gateway`

```yaml
gateway:
  platforms:
    telegram:
      enabled: true
      token: "BOT_TOKEN"
      home_channel:
        chat_id: "-100123"
        name: "Main"

  session_reset:
    mode: "both"
    at_hour: 4
    idle_minutes: 1440

  streaming:
    enabled: true
    transport: "edit"
```

### `mcp`

```yaml
mcp:
  servers:
    - name: "my-server"
      command: "npx"
      args: ["-y", "@my/mcp-server"]
      env:
        API_KEY: "..."
```

---

## 新人最常改的字段

如果你只想快速调行为，先看这些就够了：

- `model.default`
- `model.provider`
- `display.skin`
- `display.show_reasoning`
- `agent.max_turns`
- `toolsets`
- `skills.external_dirs`
- `memory.provider`
- `gateway.platforms.*`

---

## 运行时常量

与配置相关的常量集中在 `hermes_constants.py`。常见概念包括：

| 常量/概念 | 说明 |
|-----------|------|
| `HERMES_HOME` | 配置根目录 |
| `OPENROUTER_BASE_URL` | OpenRouter 默认端点 |
| reasoning levels | `low` / `medium` / `high` 等推理层级 |

---

## 环境变量覆盖

常见覆盖项：

| 环境变量 | 覆盖内容 |
|----------|----------|
| `HERMES_HOME` | 配置目录 |
| `HERMES_MODEL` | 默认模型 |
| `HERMES_BASE_URL` | API 端点 |
| `HERMES_PROVIDER` | 模型提供商 |
| `HERMES_PROFILE` | 当前 profile |

当配置“不听话”时，先检查是不是被这些变量覆盖了。

---

## 配置排障清单

当你遇到“我明明改了配置，但 Hermes 还是没按预期运行”时，按这个顺序查最省时间：

1. 确认当前 profile：你是不是用了 `--profile`
2. 确认当前环境变量：有没有 `HERMES_MODEL`、`HERMES_PROVIDER` 之类覆盖项
3. 确认改的是 `config.yaml` 还是 `.env`
4. 确认当前入口：CLI、gateway、ACP 读取路径和默认行为可能不同
5. 确认字段名仍然是当前实现里的名字，而不是旧文档里的历史名字

如果问题和工具可见性有关，再继续看：

1. `toolsets` 是否启用
2. 工具依赖的环境变量是否存在
3. 相关工具的 `check_fn` 是否通过

---

## Profiles

多 profile 支持是 Hermes 很重要的一层隔离机制。

```bash
hermes --profile work
hermes --profile personal
```

每个 profile 都有自己的：

- `config.yaml`
- `.env`
- skills
- 记忆
- session 数据

对新人最实用的理解方式是：

- profile 不是“主题”或“会话分组”
- profile 是一整套相互隔离的 Hermes 运行目录
- 模型、密钥、技能、记忆、会话都可以按 profile 完整隔离

推荐给新人的最简单用法：

- `default`：个人日常
- `work`：工作环境
- `safe`：最小权限实验环境
