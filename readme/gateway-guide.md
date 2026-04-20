# 消息网关指南

Gateway 让 Hermes 能通过 Telegram、Discord、Slack、WhatsApp、Signal 等聊天平台与用户交互，而不只是跑在本地 CLI。

如果你是新人，建议先在 CLI 把问题跑通，再来看这篇。

---

## 什么时候需要 Gateway

你只有在下面场景才需要马上读这篇：

- 要把 Hermes 部署到 Telegram / Discord / Slack 等平台
- 要调试“CLI 正常、聊天平台不正常”的问题
- 要给 Hermes 新增一个平台适配器

如果你只是改 prompt、工具、压缩或技能，通常可以先不碰 gateway。

---

## 一句话理解架构

你可以把 gateway 理解成三段式流水线：

```text
平台适配器
-> GatewayRunner
-> AIAgent
```

也就是说：

- 平台适配器负责接平台协议
- `GatewayRunner` 负责统一 session、投递、生命周期
- 真正执行工具、调模型、组织回复的还是 `AIAgent`

这能帮你快速判断问题落点：

- 收不到消息：平台适配器问题
- 会话乱串：session / runner 问题
- 回复内容不对：通常还是 agent 或配置问题

---

## 支持的平台

| 平台 | 文件 | 说明 |
|------|------|------|
| Telegram | `gateway/platforms/telegram.py` | 最适合新人先接入 |
| Discord | `gateway/platforms/discord.py` | 线程和频道较丰富 |
| Slack | `gateway/platforms/slack.py` | 团队协作场景常见 |
| WhatsApp | `gateway/platforms/whatsapp.py` | Business / bridge 场景 |
| Signal | `gateway/platforms/signal.py` | 加密消息场景 |
| Matrix | `gateway/platforms/matrix.py` | 去中心化协议 |
| Mattermost | `gateway/platforms/mattermost.py` | 自托管团队平台 |
| Email | `gateway/platforms/email.py` | 邮件收发 |
| SMS | `gateway/platforms/sms.py` | 短信投递 |
| Home Assistant | `gateway/platforms/homeassistant.py` | 家居事件入口 |
| DingTalk / Feishu / WeCom | `gateway/platforms/*.py` | 企业通信平台 |
| API Server | `gateway/platforms/api_server.py` | OpenAI 兼容接口入口 |
| Webhook | `gateway/platforms/webhook.py` | 接外部 webhook |

---

## 新人推荐：先完成一次 Telegram 闭环

原因很简单：

- 配置路径清晰
- 社区和现有实现都较成熟
- 调试反馈快

最小步骤：

1. 配置 `TELEGRAM_BOT_TOKEN`
2. 只启用 `telegram`
3. 启动 `hermes gateway start`
4. 跑 `hermes gateway status`
5. 给 bot 发一条最简单的消息

---

## 快速开始

### 1. 配置平台凭证

可以放在 `config.yaml`：

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

或者放在 `.env`：

```env
TELEGRAM_BOT_TOKEN=your_token_here
```

### 2. 启动网关

```bash
hermes gateway start
hermes gateway status
hermes gateway stop
```

### 3. 使用交互式设置

```bash
hermes setup
```

然后在 setup 里进入 Gateway 部分。

---

## 核心模块

| 模块 | 文件 | 说明 |
|------|------|------|
| `GatewayRunner` | `gateway/run.py` | 网关主编排器 |
| `SessionStore` | `gateway/session.py` | 会话管理和脱敏 |
| `Delivery` | `gateway/delivery.py` | 回复投递和流式发送 |
| `Hooks` | `gateway/hooks.py` | 网关事件钩子 |
| `Pairing` | `gateway/pairing.py` | 配对与 home channel 逻辑 |
| `Status` | `gateway/status.py` | 健康监控和锁 |

---

## 完整配置示例

```yaml
gateway:
  platforms:
    telegram:
      enabled: true
      token: "BOT_TOKEN"
      home_channel:
        chat_id: "-1001234567890"
        name: "Main Chat"

    discord:
      enabled: false
      token: null

    slack:
      enabled: false
      token: null
      app_token: null

  session_reset:
    mode: "both"          # daily / idle / both / none
    at_hour: 4
    idle_minutes: 1440

  streaming:
    enabled: true
    transport: "edit"     # edit 或 new
```

---

## 会话管理

### 自动重置策略

Gateway 支持按时间或空闲时长重置 session：

- `daily`
- `idle`
- `both`
- `none`

### PII 脱敏

`gateway/session.py` 会对用户标识进行脱敏和内部映射，所以你在日志或调试输出里看到的未必是平台原始 ID。

---

## 定时任务投递

如果你配置了 `home_channel`，cron 结果可以自动投递到目标平台。

这意味着：

- cron 是异步执行器
- gateway 是消息投递出口

出现“任务跑了但消息没发出来”的问题时，要同时查 cron 和 gateway。

---

## Docker 部署

生产环境常用方式：

```bash
docker build -t hermes-agent .
docker run -d \
  --name hermes-gateway \
  --env-file ~/.hermes/.env \
  -v ~/.hermes:/root/.hermes \
  hermes-agent \
  hermes gateway start
```

---

## 调试清单

gateway 有问题时，推荐按这个顺序排查：

1. `source venv/bin/activate`
2. `hermes doctor`
3. `hermes gateway status`
4. 只启用一个平台，降低变量数量
5. 在 CLI 里复现相同 prompt，确认不是 agent 核心问题
6. 看 `gateway/run.py` 和对应平台适配器

常见症状对应的排查方向：

- 收不到消息：token / webhook / 平台适配器
- 会话串了：`gateway/session.py`
- 回复发不出去：`gateway/delivery.py`
- 重置时机不对：`session_reset`
- 定时消息不投递：`home_channel` 与 cron

---

## 新增平台适配器

想支持一个新平台时，核心步骤通常是：

1. 在 `gateway/platforms/` 下新增文件
2. 继承平台抽象基类
3. 实现启动、停止、接收和发送逻辑
4. 在网关配置和平台枚举中注册
5. 在 `tests/gateway/` 下补测试

更具体的开发流程请看 [development-guide.md](./development-guide.md)。
