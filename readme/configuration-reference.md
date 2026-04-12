# 配置参考

Hermes Agent 的所有配置存放在 `~/.hermes/` 目录下。本文档列出所有配置项及其说明。

---

## 目录结构

```
~/.hermes/
├── config.yaml          # 主配置文件
├── .env                 # API 密钥和环境变量
├── skills/              # 所有活跃技能
├── memories/            # 持久化记忆
│   ├── MEMORY.md        # 智能体记忆
│   └── USER.md          # 用户信息
├── state.db             # SQLite 会话数据库
├── sessions/            # 会话 JSON 日志
├── cron/                # 定时任务数据
├── cache/               # Token/元数据缓存
└── profiles/            # 多配置文件
```

---

## .env — 环境变量

存放所有 API 密钥和敏感配置：

```env
# ===== LLM 提供商 =====
OPENROUTER_API_KEY=sk-or-...        # OpenRouter（支持 100+ 模型）
ANTHROPIC_API_KEY=sk-ant-...         # Anthropic Claude
OPENAI_API_KEY=sk-...                # OpenAI
NOUS_API_KEY=...                     # Nous Research Portal

# ===== 工具 API =====
EXA_API_KEY=...                      # Exa 网页搜索
FIRECRAWL_API_KEY=...                # Firecrawl 网页抓取
FAL_KEY=...                          # FAL 图像生成
ELEVENLABS_API_KEY=...               # ElevenLabs TTS

# ===== 消息平台 =====
TELEGRAM_BOT_TOKEN=...               # Telegram
DISCORD_BOT_TOKEN=...                # Discord
SLACK_BOT_TOKEN=...                  # Slack
SLACK_APP_TOKEN=...                  # Slack App

# ===== 其他 =====
HOMEASSISTANT_TOKEN=...              # Home Assistant
GITHUB_TOKEN=...                     # GitHub（可选）
```

---

## config.yaml — 主配置文件

### model — 模型配置

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"  # 默认模型
  base_url: null                            # 自定义 API 端点
  provider: "openrouter"                    # 提供商：openrouter/anthropic/openai/custom
```

### terminal — 终端配置

```yaml
terminal:
  env_type: "local"          # 执行环境：local/docker/ssh/modal/daytona/singularity
  cwd: null                  # 工作目录（默认当前目录）
  timeout: 120               # 命令超时（秒）

  # Docker 特定
  docker_image: "ubuntu:22.04"
  docker_volumes: []

  # SSH 特定
  ssh_host: null
  ssh_user: null
  ssh_key: null

  # Modal 特定
  modal_app_name: null
```

### browser — 浏览器配置

```yaml
browser:
  inactivity_timeout: 300    # 浏览器空闲超时（秒）
  record_sessions: false     # 是否录制浏览器会话
  provider: "browser_use"    # 提供商：browser_use/browserbase/firecrawl
```

### compression — 上下文压缩

```yaml
compression:
  enabled: true              # 是否启用自动压缩
  threshold: 0.5             # 触发压缩的 token 使用率阈值（0.0~1.0）
  summary_model: null        # 用于生成摘要的模型（默认同主模型）
```

### smart_model_routing — 智能路由

```yaml
smart_model_routing:
  enabled: false             # 是否启用智能模型路由
  cheap_model: "openai/gpt-4o-mini"  # 简单任务使用的便宜模型
```

### agent — 智能体行为

```yaml
agent:
  max_turns: 50              # 单次对话最大轮数
  system_prompt: null        # 自定义系统提示词（追加到默认之后）
  personalities: {}          # 自定义人格
  # 示例：
  # personalities:
  #   researcher:
  #     prompt: "You are a meticulous research assistant..."
  #   creative:
  #     prompt: "You are a creative writing expert..."
```

### display — 显示设置

```yaml
display:
  compact: false             # 紧凑模式
  skin: "default"            # 主题皮肤
  streaming: true            # 流式输出
  show_reasoning: true       # 显示推理过程
```

### delegation — 子智能体配置

```yaml
delegation:
  max_iterations: 20         # 子智能体最大迭代次数
  default_toolsets:           # 子智能体默认工具集
    - terminal
    - file
    - web
  model: null                # 子智能体使用的模型（默认同主模型）
  base_url: null             # 子智能体的 API 端点
```

### toolsets — 工具集配置

```yaml
toolsets:
  enabled:                   # 启用的工具集
    - web
    - terminal
    - file
    - skills
    - memory
    - cron
    - delegate
  disabled:                  # 禁用的工具集
    - browser
    - vision
```

### skills — 技能配置

```yaml
skills:
  disabled:                  # 禁用的技能
    - social-media/xitter
  external_dirs:             # 额外技能目录
    - /path/to/custom/skills
```

### memory — 记忆配置

```yaml
memory:
  provider: "builtin"        # 记忆提供商：builtin/mem0/honcho/...
  nudge_interval: 10         # 记忆提醒间隔（对话轮数）
```

### gateway — 网关配置

```yaml
gateway:
  platforms:
    telegram:
      enabled: true
      token: "BOT_TOKEN"
      home_channel:
        chat_id: "-100123"
        name: "Main"
    discord:
      enabled: false
      token: null
    slack:
      enabled: false
      token: null
      app_token: null

  session_reset:
    mode: "both"             # daily/idle/both/none
    at_hour: 4
    idle_minutes: 1440

  streaming:
    enabled: true
    transport: "edit"
```

### mcp — MCP 服务器配置

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

## 运行时常量（`hermes_constants.py`）

| 常量 | 说明 |
|------|------|
| `HERMES_HOME` | 配置目录（默认 `~/.hermes`，可通过 `HERMES_HOME` 环境变量覆盖） |
| `OPENROUTER_BASE_URL` | OpenRouter API 端点 |
| Reasoning Levels | `low` / `medium` / `high` — 推理深度级别 |

---

## 环境变量覆盖

大多数配置项可以通过环境变量覆盖：

| 环境变量 | 覆盖的配置 |
|----------|-----------|
| `HERMES_HOME` | 配置目录路径 |
| `HERMES_MODEL` | 默认模型 |
| `HERMES_BASE_URL` | API 端点 |
| `HERMES_PROVIDER` | 提供商 |
| `HERMES_PROFILE` | 配置文件名 |

---

## 多 Profile 支持

每个 Profile 是 `~/.hermes/` 下的一个独立子目录：

```bash
hermes --profile work      # 使用 ~/.hermes/profiles/work/
hermes --profile personal   # 使用 ~/.hermes/profiles/personal/
```

每个 Profile 有自己的 `config.yaml`、`.env`、技能、记忆等，完全隔离。
