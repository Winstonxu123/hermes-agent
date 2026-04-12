# 消息网关指南

消息网关（Gateway）让 Hermes Agent 能够通过 Telegram、Discord、Slack 等聊天平台与用户交互，而不仅限于本地终端。

---

## 支持的平台

| 平台 | 文件 | 特性 |
|------|------|------|
| **Telegram** | `gateway/platforms/telegram.py` | 语音备忘录、群组对话、DM 配对 |
| **Discord** | `gateway/platforms/discord.py` | 线程、语音频道、角色控制 |
| **Slack** | `gateway/platforms/slack.py` | 线程回复、格式化 |
| **WhatsApp** | `gateway/platforms/whatsapp.py` | WhatsApp Business API |
| **Signal** | `gateway/platforms/signal.py` | 端到端加密、群组 |
| **Matrix** | `gateway/platforms/matrix.py` | 去中心化协议 |
| **Mattermost** | `gateway/platforms/mattermost.py` | 自托管团队协作 |
| **Email** | `gateway/platforms/email.py` | 邮件收发 |
| **SMS** | `gateway/platforms/sms.py` | 短信 |
| **Home Assistant** | `gateway/platforms/homeassistant.py` | 智能家居 |
| **DingTalk** | `gateway/platforms/dingtalk.py` | 钉钉 |
| **Feishu** | `gateway/platforms/feishu.py` | 飞书 |
| **WeChat** | `gateway/platforms/wecom.py` | 企业微信 |
| **API Server** | `gateway/platforms/api_server.py` | REST API |
| **Webhook** | `gateway/platforms/webhook.py` | Webhook 监听 |

---

## 快速开始

### 1. 配置平台凭证

编辑 `~/.hermes/config.yaml`：

```yaml
gateway:
  platforms:
    telegram:
      enabled: true
      token: "YOUR_TELEGRAM_BOT_TOKEN"
      home_channel:
        chat_id: "YOUR_CHAT_ID"
        name: "My Chat"
```

或在 `~/.hermes/.env` 中设置：

```env
TELEGRAM_BOT_TOKEN=your_token_here
```

### 2. 启动网关

```bash
# 启动所有已启用的平台
hermes gateway start

# 查看状态
hermes gateway status

# 停止
hermes gateway stop
```

### 3. 使用交互式设置

```bash
hermes setup
# 选择 "Gateway" 部分进行配置
```

---

## 架构概览

```
外部平台（Telegram/Discord/...）
    │
    ▼
平台适配器（gateway/platforms/*.py）
    │  - 接收消息
    │  - 转换为统一格式
    │
    ▼
GatewayRunner（gateway/run.py）
    │  - 管理所有平台的生命周期
    │  - 创建/恢复会话
    │
    ▼
AIAgent（run_agent.py）
    │  - 处理消息
    │  - 执行工具
    │  - 生成回复
    │
    ▼
消息投递（gateway/delivery.py）
    │  - 格式化回复
    │  - 流式推送
    │
    ▼
外部平台（返回给用户）
```

### 核心模块

| 模块 | 文件 | 说明 |
|------|------|------|
| `GatewayRunner` | `gateway/run.py` | 网关主入口，管理平台生命周期 |
| `Session` | `gateway/session.py` | 会话状态管理、PII 脱敏 |
| `Config` | `gateway/config.py` | 网关配置加载与校验 |
| `Delivery` | `gateway/delivery.py` | 消息投递编排 |
| `Hooks` | `gateway/hooks.py` | 事件钩子系统 |
| `Pairing` | `gateway/pairing.py` | DM 配对工作流 |
| `Status` | `gateway/status.py` | 健康监控 |

---

## 完整配置示例

```yaml
gateway:
  # 平台配置
  platforms:
    telegram:
      enabled: true
      token: "BOT_TOKEN"
      home_channel:
        chat_id: "-1001234567890"
        name: "Main Chat"

    discord:
      enabled: true
      token: "DISCORD_BOT_TOKEN"

    slack:
      enabled: false
      token: "xoxb-..."
      app_token: "xapp-..."

  # 会话重置策略
  session_reset:
    mode: "both"          # daily / idle / both / none
    at_hour: 4            # 每天几点重置（mode=daily 或 both）
    idle_minutes: 1440    # 空闲多少分钟重置（mode=idle 或 both）

  # 流式输出
  streaming:
    enabled: true
    transport: "edit"     # edit（编辑消息）或 new（发新消息）
```

---

## 会话管理

### 会话重置策略

网关支持自动重置长时间不活跃的会话：

- `daily` — 每天在指定时刻（`at_hour`）重置
- `idle` — 空闲超过 `idle_minutes` 后重置
- `both` — 同时启用以上两种
- `none` — 永不自动重置

### PII 脱敏

`gateway/session.py` 自动对用户 ID 和聊天 ID 进行哈希处理：

- 用户 ID → `user_<12位hex>`
- 聊天 ID → `platform:<hash>`（保留平台前缀）
- 电话号码 → E.164 格式掩码

---

## 定时任务投递

定时任务（Cron）可以将结果自动投递到网关平台：

```yaml
# 在配置中指定 home_channel 后，cron 结果会发到该频道
gateway:
  platforms:
    telegram:
      home_channel:
        chat_id: "-1001234567890"
```

通过 `/cron` 命令或 `hermes cron` 创建定时任务时，可以指定投递目标。

---

## 事件钩子

`gateway/hooks.py` 提供事件钩子系统，可以在特定事件触发时执行自定义逻辑。

内置钩子：
- `boot_md.py` — 启动时加载 `SOUL.md` 人格文件

---

## Docker 部署

使用 Docker 部署网关是推荐的生产方式：

```bash
docker build -t hermes-agent .
docker run -d \
  --name hermes-gateway \
  --env-file ~/.hermes/.env \
  -v ~/.hermes:/root/.hermes \
  hermes-agent \
  hermes gateway start
```

`docker/entrypoint.sh` 和 `docker/SOUL.md` 提供了默认的容器启动配置。

---

## 添加新平台

想为新的聊天平台添加支持？参见 [开发者指南](./development-guide.md#添加新的消息平台)。

核心步骤：
1. 在 `gateway/platforms/` 下创建新文件
2. 继承 `base.py` 中的抽象基类
3. 实现 `start()`、`stop()`、`send_message()` 等方法
4. 在 `gateway/config.py` 的 `Platform` 枚举中注册
5. 编写测试
