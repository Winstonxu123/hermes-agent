# `_build_system_prompt` 方法详细分析

> 源码位置：`run_agent.py` L3092-3257（约 165 行）
> 作用：装配一份发给 LLM 的 **system prompt**（系统提示词），告诉模型"你是谁、你能做什么、你现在在哪里"。

---

## 阅读前提

推荐先读：

1. [simple.md](./simple.md)
2. [architecture.md](./architecture.md)
3. [run_agent-reading-map.md](./run_agent-reading-map.md)

再看这篇。这样你会更容易理解 `_build_system_prompt()` 在整个 agent 生命周期里的位置，而不是把它当成一段孤立的大字符串拼接代码。

---

## 一句话理解

> **把 12 块零件（身份 + 行为指令 + 记忆 + 技能 + 上下文文件 + 时间 + 环境 + 平台...）按顺序叠在一起，用 `\n\n` 连接成一个超大字符串。**

这个字符串会塞进每次 API 请求的第一条消息（`role=system`），LLM 会把它当作"最高优先级的说明书"来遵循。

---

## 1. 方法签名和职责

```python
def _build_system_prompt(self, system_message: str = None) -> str:
```

| 入参 | 含义 |
|---|---|
| `system_message` | **调用方追加的额外提示**（可选）。比如子 agent 被 delegate 时，主 agent 可以额外注入一段"你这次的任务是 XX"。日常会话中为 `None` |

**返回**：一个完整的多段文本，类似：

```
You are Hermes Agent, an intelligent AI assistant created by Nous Research...

You have persistent memory across sessions. Save durable facts...

<user-profile>
user is a data scientist...
</user-profile>

# Available Skills
- debug-react: Troubleshooting React hook issues
- deploy-cloudrun: Deploying to Google Cloud Run
...

(from AGENTS.md)
This repo uses pnpm, not npm. Test command is `pnpm test`...

Conversation started: Wednesday, April 18, 2026 02:30 PM
Session ID: abc123
Model: claude-opus-4.7
Provider: anthropic

You are on a text messaging communication platform, Telegram...
```

---

## 2. 调用与缓存时机（非常关键）

这个方法**不是每轮对话都调一次**，而是**每个会话只构建一次**，缓存在 `self._cached_system_prompt`。

### 调用点总览（源码行号）

| 位置 | 触发条件 | 行为 |
|---|---|---|
| `run_agent.py:7829-7845` | 本次会话第一次进入 `run_conversation` | 懒加载：如果缓存空，调用一次并写入缓存 |
| `run_agent.py:6753-6755` | 上下文压缩完毕 | 丢弃缓存并**重建**（因为身份/记忆可能变了） |
| `run_agent.py:3419-3426` | `_invalidate_system_prompt()` | 清空缓存，强制下次重建 |
| `run_agent.py:1605` | `switch_model()` 切模型 | 清空缓存（新模型可能需要不同指令） |
| `run_agent.py:7864` | 首次构建后 | 写入 SQLite，下次进程重启可恢复 |
| `run_agent.py:6612-6613` | 每次 API 调用前 | 把缓存字符串塞到 messages 列表最前 |

### 为什么要缓存？

> **为了吃到"前缀缓存"（Prefix Cache）的红利。**

LLM 提供商（Anthropic、OpenAI 等）对**完全相同的输入前缀**会走缓存、便宜一半。system prompt 是每轮对话的第一条消息，如果每次都不同，整个缓存就失效了。所以 Hermes Agent 把时间戳（"Conversation started..."）**冻结在首次构建的那一刻**，后面 10 轮、20 轮对话都用同一份 system prompt。

这也是为什么 L3097 的 docstring 写：
> *Called once per session (cached on `self._cached_system_prompt`) and only rebuilt after context compression events. This ensures the system prompt is stable across all turns in a session, maximizing prefix cache hits.*

---

## 3. 7 层结构（docstring 宣称的）

源码开头 L3100-3107 写了官方"层序"：

```
1. Agent identity           — SOUL.md 或 DEFAULT_AGENT_IDENTITY
2. User / gateway prompt    — 调用方传进来的 system_message
3. Persistent memory        — 持久化记忆快照
4. Skills guidance          — 技能指引（如果启用了 skills 工具）
5. Context files            — AGENTS.md, .cursorrules 等
6. Current date & time      — 冻结在构建时刻
7. Platform-specific hint   — 平台定制指引（WhatsApp/Telegram 等）
```

**但实际代码里插入了比 7 层更多的内容**（至少 12 类）。下面按真实执行顺序拆解。

---

## 4. 真实执行顺序（12 步详解）

### 🟦 步骤 1 — Agent 身份（L3109-3119）

```python
if not self.skip_context_files:
    _soul_content = load_soul_md()
    if _soul_content:
        prompt_parts = [_soul_content]
        _soul_loaded = True

if not _soul_loaded:
    prompt_parts = [DEFAULT_AGENT_IDENTITY]
```

- **优先用 `SOUL.md`** — 用户在项目根目录或 `~/.hermes/` 下放一个 `SOUL.md`，里面自定义 Agent 的人格（来自 `agent/prompt_builder.py:load_soul_md()`）
- **否则用默认身份** — `DEFAULT_AGENT_IDENTITY`（`agent/prompt_builder.py:L134`），内容：

  > "You are Hermes Agent, an intelligent AI assistant created by Nous Research. You are helpful, knowledgeable, and direct..."

- `skip_context_files` 为 True 时跳过（用于纯工具模式，不读项目文件）

**关键变量**：`_soul_loaded`（bool）后面会用来决定要不要在步骤 9 里排除 SOUL.md。

---

### 🟦 步骤 2 — 工具感知的行为指引（L3121-3134）

```python
tool_guidance = []
if "memory" in self.valid_tool_names:
    tool_guidance.append(MEMORY_GUIDANCE)
if "session_search" in self.valid_tool_names:
    tool_guidance.append(SESSION_SEARCH_GUIDANCE)
if "skill_manage" in self.valid_tool_names:
    tool_guidance.append(SKILLS_GUIDANCE)
if tool_guidance:
    prompt_parts.append(" ".join(tool_guidance))

nous_subscription_prompt = build_nous_subscription_prompt(self.valid_tool_names)
if nous_subscription_prompt:
    prompt_parts.append(nous_subscription_prompt)
```

**逻辑**：只有当某个工具**真的被加载了**，才告诉模型"你有这个工具、请这样用"。

| 条件 | 注入内容 | 一句话描述 |
|---|---|---|
| 工具 `memory` 可用 | `MEMORY_GUIDANCE` | 教模型怎么存/不存记忆（只存用户偏好，不存任务进度） |
| 工具 `session_search` 可用 | `SESSION_SEARCH_GUIDANCE` | 教模型"用户提到过去的事时，先搜历史对话" |
| 工具 `skill_manage` 可用 | `SKILLS_GUIDANCE` | 教模型"5+ 工具调用的任务，完成后存成技能" |
| Nous 订阅用户 | `build_nous_subscription_prompt` | Nous Portal 专属提示 |

⚠️ **设计哲学**：**不要告诉模型它没有的能力**。如果你禁用了 memory 工具却告诉它"你有记忆"，模型会编造 tool_call 然后失败。

---

### 🟦 步骤 3 — 工具使用强制（L3135-3167）

```python
if self.valid_tool_names:
    _enforce = self._tool_use_enforcement
    # "auto" / True / False / list 四种配置模式
    if _inject:
        prompt_parts.append(TOOL_USE_ENFORCEMENT_GUIDANCE)
        if "gemini" in _model_lower or "gemma" in _model_lower:
            prompt_parts.append(GOOGLE_MODEL_OPERATIONAL_GUIDANCE)
        if "gpt" in _model_lower or "codex" in _model_lower:
            prompt_parts.append(OPENAI_MODEL_EXECUTION_GUIDANCE)
```

**要解决的问题**：某些模型（尤其是 GPT、Gemini、Grok）喜欢**"描述自己要做什么"而不是真的去做**。例如说 "我会帮你检查文件" 但不真的 call read_file 工具。

**三段内容**：

| 注入项 | 触发条件 | 作用 |
|---|---|---|
| `TOOL_USE_ENFORCEMENT_GUIDANCE` | 模型名匹配 `("gpt", "codex", "gemini", "gemma", "grok")` | 强制"说要做就立刻 call tool" |
| `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` | 模型名含 `gemini`/`gemma` | Google 模型专项：用绝对路径、检查依赖、并行调用、保持简洁 |
| `OPENAI_MODEL_EXECUTION_GUIDANCE` | 模型名含 `gpt`/`codex` | GPT 专项：工具持久性、必须用工具的场景、行动优先于提问 |

**配置开关**（`config.yaml` → `agent.tool_use_enforcement`）：

| 值 | 效果 |
|---|---|
| `"auto"`（默认） | 匹配 `TOOL_USE_ENFORCEMENT_MODELS` 列表时注入 |
| `true` | 所有模型都注入 |
| `false` | 永不注入 |
| `list`（如 `["claude", "grok"]`） | 用户自定义匹配串 |

---

### 🟦 步骤 4 — 调用方传入的 system_message（L3172-3173）

```python
if system_message is not None:
    prompt_parts.append(system_message)
```

**谁用这个**：
- 子 agent 被 `delegate_task` 调起时，父 agent 传入 "你这次的角色是..."
- 测试代码手动注入 prompt

⚠️ **注意**：`self.ephemeral_system_prompt` **不在这里** 处理。它是每次 API 调用时临时贴在前面的（见 `_build_api_kwargs`），**不进入缓存**——所以改它不会失效 prefix cache。

---

### 🟦 步骤 5 — 持久化记忆（内置 memory store）（L3175-3184）

```python
if self._memory_store:
    if self._memory_enabled:
        mem_block = self._memory_store.format_for_system_prompt("memory")
        if mem_block:
            prompt_parts.append(mem_block)
    if self._user_profile_enabled:
        user_block = self._memory_store.format_for_system_prompt("user")
        if user_block:
            prompt_parts.append(user_block)
```

**两个独立块**：

| 块 | 来自文件 | 典型内容 |
|---|---|---|
| `memory_block` | `~/.hermes/memory/*.md` | 用户偏好、环境怪癖、历史教训。**可被 memory 工具读写** |
| `user_block` | `~/.hermes/memory/USER.md` | 用户档案（姓名、角色、偏好）。**总是注入，不受 memory_enabled 影响** |

两个块通常被 `<memory>...</memory>` 或 `<user-profile>...</user-profile>` 标签包裹。

---

### 🟦 步骤 6 — 外部记忆提供商（plugin）（L3187-3193）

```python
if self._memory_manager:
    try:
        _ext_mem_block = self._memory_manager.build_system_prompt()
        if _ext_mem_block:
            prompt_parts.append(_ext_mem_block)
    except Exception:
        pass
```

**与步骤 5 的区别**：步骤 5 是**内置**的基于文件的记忆；步骤 6 是**插件式**的外部记忆提供商（如 Mem0、Letta、Zep 等向量数据库记忆）。

**容错**：整段用 `try/except` 兜住，外部服务挂了也不会影响 prompt 构建。

---

### 🟦 步骤 7 — 技能索引（L3195-3211）

```python
has_skills_tools = any(name in self.valid_tool_names 
                       for name in ['skills_list', 'skill_view', 'skill_manage'])
if has_skills_tools:
    avail_toolsets = {...}
    skills_prompt = build_skills_system_prompt(
        available_tools=self.valid_tool_names,
        available_toolsets=avail_toolsets,
    )
```

**生成内容**：一个列表，告诉模型当前可用哪些技能（skill 的名字 + 一句话描述）。模型自己决定要不要 `skill_view(name)` 去看完整内容。

**为什么要列清单**：技能文件可能有几百个，总字数几十万 token。所以 Hermes 只给**索引**，详细内容按需加载。

---

### 🟦 步骤 8 — 项目上下文文件（L3213-3222）

```python
if not self.skip_context_files:
    _context_cwd = os.getenv("TERMINAL_CWD") or None
    context_files_prompt = build_context_files_prompt(
        cwd=_context_cwd, skip_soul=_soul_loaded)
    if context_files_prompt:
        prompt_parts.append(context_files_prompt)
```

**读取的文件**（`agent/prompt_builder.py:build_context_files_prompt`）：

| 文件名 | 用途 |
|---|---|
| `AGENTS.md` | 项目通用 agent 约定（推荐） |
| `.hermes.md` | Hermes 专用项目约定 |
| `CLAUDE.md` | Claude Code 约定 |
| `.cursorrules` | Cursor 编辑器约定 |
| `SOUL.md` | Agent 人格（**如果已经在步骤 1 用过，这里 skip**） |

**⚡ 工作目录陷阱**：
- 读取 `TERMINAL_CWD` 环境变量而**非 `os.getcwd()`**
- **原因**：当 Hermes 作为 gateway 服务运行时，进程的 cwd 是安装目录（比如 `/opt/hermes-agent`），读到的会是**框架自己的 AGENTS.md**，污染用户 prompt 并浪费约 10k token

---

### 🟦 步骤 9 — 时间戳（L3224-3233）

```python
from hermes_time import now as _hermes_now
now = _hermes_now()
timestamp_line = f"Conversation started: {now.strftime('%A, %B %d, %Y %I:%M %p')}"
if self.pass_session_id and self.session_id:
    timestamp_line += f"\nSession ID: {self.session_id}"
if self.model:
    timestamp_line += f"\nModel: {self.model}"
if self.provider:
    timestamp_line += f"\nProvider: {self.provider}"
prompt_parts.append(timestamp_line)
```

**内容举例**：
```
Conversation started: Wednesday, April 18, 2026 02:30 PM
Session ID: abc-123-def
Model: claude-opus-4.7
Provider: anthropic
```

**关键设计**：
1. **时间冻结** — 在首次构建时 `now()` 取一次，之后整个会话不变。保证 prefix cache 命中。
2. **不是精确时间** — 要精确时间必须 call `terminal` 工具 `date`（OpenAI 模型指引里明确要求）
3. **模型/provider 写入 prompt** — 帮助模型知道"自己是谁"（某些模型的 API 不会在 response 里返回这些字段）

---

### 🟦 步骤 10 — 阿里云 Workaround（L3235-3245）

```python
if self.provider == "alibaba":
    _model_short = self.model.split("/")[-1] if "/" in self.model else self.model
    prompt_parts.append(
        f"You are powered by the model named {_model_short}. "
        f"The exact model ID is {self.model}. "
        f"When asked what model you are, always answer based on this information, "
        f"not on any model name returned by the API."
    )
```

**专治一个 API bug**：阿里云 Coding Plan 无论你请求什么模型，API 返回的 model 字段**总是 `glm-4.7`**。所以要显式告诉模型"别信 API 返回的名字，以我说的为准"。

这是**一个典型的"供应商特殊处理"**，整个文件里有很多这类补丁。

---

### 🟦 步骤 11 — 运行环境提示（L3247-3251）

```python
_env_hints = build_environment_hints()
if _env_hints:
    prompt_parts.append(_env_hints)
```

**探测内容**（`agent/prompt_builder.py:build_environment_hints`）：

| 场景 | 注入提示 |
|---|---|
| WSL | "你在 WSL 里，`C:\` 要用 `/mnt/c/` 来访问" |
| Termux（Android） | "你在 Android Termux 里，没有 systemd、`/data/data/com.termux/...`" |
| Docker 容器 | "你在容器里，文件修改出容器后会丢" |

---

### 🟦 步骤 12 — 聊天平台定制（L3253-3255）

```python
platform_key = (self.platform or "").lower().strip()
if platform_key in PLATFORM_HINTS:
    prompt_parts.append(PLATFORM_HINTS[platform_key])
```

`PLATFORM_HINTS`（`agent/prompt_builder.py:L285`）是一个 dict，key 是平台名，value 是对应的注意事项：

| 平台 | 核心提示 |
|---|---|
| `whatsapp` | 不要用 markdown；用 `MEDIA:/path/to/file` 发附件 |
| `telegram` | 同上，支持 `.ogg` 语音、`.mp4` 视频 |
| `discord` | 附件用 `MEDIA:`；支持 markdown |
| `slack` | 附件用 `MEDIA:`；slack 专用链接格式 |
| `signal` | 不要 markdown；附件支持 |
| `email` | 写得像邮件；不要问候和签名（会重复） |
| `cli`（默认） | 不触发任何额外提示 |

⚠️ **gateway 模式下这一步很关键**——不加这个，模型会在 WhatsApp 发一堆 markdown 语法乱码给用户。

---

### 🔚 最后：拼接（L3257）

```python
return "\n\n".join(p.strip() for p in prompt_parts if p.strip())
```

- **去空格** `.strip()`
- **过滤空串** `if p.strip()`
- **段间用两个换行** `\n\n` — 保证 markdown 段落边界清晰

---

## 5. 组装流程图（直观版）

```
                   ┌──────────────────────┐
                   │  _build_system_prompt │
                   └──────────┬───────────┘
                              │
    ┌─────────────────────────┼─────────────────────────────┐
    │ 1. 身份                  │                             │
    │    SOUL.md 优先，否则 DEFAULT_AGENT_IDENTITY           │
    │                                                       │
    │ 2. 工具指引（只注入实际加载的工具）                    │
    │    memory / session_search / skills / nous_subscription│
    │                                                       │
    │ 3. 工具强制（只针对需要的模型）                        │
    │    ├─ 通用：TOOL_USE_ENFORCEMENT                       │
    │    ├─ Google：GOOGLE_MODEL_OPERATIONAL                 │
    │    └─ OpenAI：OPENAI_MODEL_EXECUTION                   │
    │                                                       │
    │ 4. 调用方的 system_message（delegate 或测试用）        │
    │                                                       │
    │ 5. 内置记忆（memory_store）                            │
    │    ├─ memory block                                     │
    │    └─ user profile                                     │
    │                                                       │
    │ 6. 外部记忆（memory_manager plugin）                   │
    │                                                       │
    │ 7. 技能索引（build_skills_system_prompt）              │
    │                                                       │
    │ 8. 项目上下文文件                                      │
    │    AGENTS.md / .hermes.md / CLAUDE.md / .cursorrules   │
    │                                                       │
    │ 9. 时间戳 + session_id + model + provider              │
    │                                                       │
    │ 10. 阿里云 workaround（只有 provider=alibaba）         │
    │                                                       │
    │ 11. 环境提示（WSL / Termux / 容器）                    │
    │                                                       │
    │ 12. 平台定制（whatsapp / telegram / slack / ...）      │
    └─────────────────────────┬─────────────────────────────┘
                              │
                   "\n\n".join(非空部分)
                              │
                              ▼
                     return 完整 system prompt
```

---

## 6. 关键设计决策

### ✅ 决策 1：缓存 + 冻结

- **首次构建后缓存**在 `self._cached_system_prompt`
- **时间戳冻结**在构建时刻
- **只在压缩/切模型后重建**
- 目的：**最大化 prefix cache 命中率**（API 费用减半）

### ✅ 决策 2：按需注入，不"大而全"

- 工具指引只在对应工具存在时注入
- 模型专项指引只在对应模型家族下注入
- 平台指引只在非 CLI 平台下注入
- 目的：**省 token + 避免误导模型说它有没有的能力**

### ✅ 决策 3：外部文件分层

- `SOUL.md` 第一优先（角色人格）
- `AGENTS.md`/`CLAUDE.md` 次优（项目约定）
- 二者不重叠（SOUL 用过就不再作为 context file）
- 目的：**项目级 + 用户级的两层覆盖**

### ✅ 决策 4：供应商 workaround 内嵌

- 阿里云模型名 bug → 显式声明
- 非 ASCII 字符 → 清洗
- OpenAI developer role → API 时转换（不在这个方法里）
- 目的：**封装供应商差异，上层感知不到**

---

## 7. 有哪些外部依赖？

这个方法本身只做"胶水"工作，真正的内容在 `agent/prompt_builder.py` 里：

| 被调用的外部符号 | 定义位置 | 作用 |
|---|---|---|
| `DEFAULT_AGENT_IDENTITY` | `agent/prompt_builder.py:L134` | 硬编码默认身份 |
| `MEMORY_GUIDANCE` | `agent/prompt_builder.py:L144` | 记忆使用指引 |
| `SESSION_SEARCH_GUIDANCE` | `agent/prompt_builder.py:L158` | 历史搜索指引 |
| `SKILLS_GUIDANCE` | `agent/prompt_builder.py:L164` | 技能使用指引 |
| `TOOL_USE_ENFORCEMENT_GUIDANCE` | `agent/prompt_builder.py:L173` | 工具强制模板 |
| `TOOL_USE_ENFORCEMENT_MODELS` | `agent/prompt_builder.py:L190` | `("gpt", "codex", "gemini", "gemma", "grok")` |
| `OPENAI_MODEL_EXECUTION_GUIDANCE` | `agent/prompt_builder.py:L196` | GPT/Codex 专项纪律 |
| `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` | `agent/prompt_builder.py:L258` | Gemini/Gemma 专项纪律 |
| `PLATFORM_HINTS` | `agent/prompt_builder.py:L285` | 平台 dict |
| `load_soul_md()` | `agent/prompt_builder.py:L885` | 读取 SOUL.md |
| `build_skills_system_prompt()` | `agent/prompt_builder.py:L575` | 生成技能索引 |
| `build_context_files_prompt()` | `agent/prompt_builder.py:L998` | 读取 AGENTS.md 等 |
| `build_environment_hints()` | `agent/prompt_builder.py:L399` | WSL/Termux 探测 |
| `build_nous_subscription_prompt()` | `agent/prompt_builder.py:L803` | Nous 订阅提示 |
| `get_toolset_for_tool()` | `model_tools.py` | 根据工具名查其工具集 |
| `_hermes_now()` | `hermes_time.py` | 可被 mock 的时间源（测试用） |

---

## 8. 常见 debug 技巧

### Q1：我想看实际发送给模型的 system prompt？

```python
agent = AIAgent(...)
agent.run_conversation("hi")  # 首次构建会写入缓存
print(agent._cached_system_prompt)
```

或查看 SQLite：
```bash
sqlite3 ~/.hermes/sessions.db "SELECT system_prompt FROM sessions ORDER BY created_at DESC LIMIT 1;"
```

### Q2：我改了 SOUL.md，但模型行为没变？

因为缓存还在用旧的。**解决**：
- 新开一个会话（session_id 改变）
- 或调 `agent._invalidate_system_prompt()` 手动失效

### Q3：我启了 memory 工具，但 prompt 里没有 MEMORY_GUIDANCE？

检查 `self.valid_tool_names` —— 工具集里 "memory" 真的被加载了吗？

```python
print("memory" in agent.valid_tool_names)
```

### Q4：怎么测试某个模型触发了哪些 guidance？

临时把 `self._tool_use_enforcement` 设置为 True，看输出哪些段：

```python
agent._tool_use_enforcement = True
agent._invalidate_system_prompt()
prompt = agent._build_system_prompt()
print(prompt)
```

---

## 9. 如果要拆分这个方法，怎么拆？

这个方法其实已经是"调度层"了，每个步骤都调外部函数做重活。但 165 行仍然偏长。**建议**：

```python
# 重构后的骨架
def _build_system_prompt(self, system_message: str = None) -> str:
    parts = []
    parts.extend(self._build_identity_section())      # 步骤 1
    parts.extend(self._build_tool_guidance_section()) # 步骤 2-3
    if system_message:                                # 步骤 4
        parts.append(system_message)
    parts.extend(self._build_memory_section())        # 步骤 5-6
    parts.extend(self._build_skills_section())        # 步骤 7
    parts.extend(self._build_context_files_section()) # 步骤 8
    parts.append(self._build_timestamp_line())        # 步骤 9
    parts.extend(self._build_provider_workarounds())  # 步骤 10
    parts.extend(self._build_environment_section())   # 步骤 11-12
    return "\n\n".join(p.strip() for p in parts if p.strip())
```

每个 `_build_*_section` 约 10-25 行，单一职责，单测好写。但这是可选重构，当前代码读起来也算清晰。

---

## 10. 一句话收尾

> **`_build_system_prompt` 不是"一个函数"，是整个 Agent 如何自我介绍的 SOP：  
> 把 `身份 + 工具 + 记忆 + 技能 + 项目 + 时间 + 环境 + 平台` 这八类信息，按"该给才给、给一次就不再改"的原则，叠成一份给 LLM 的说明书。**
