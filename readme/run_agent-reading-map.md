# `run_agent.py` 导读地图（小白版）

> 这个文件是 Hermes Agent 的心脏，**10,800 行**，其中 99% 是一个超大类 `AIAgent`（约 100 个方法）。
> 本文档不改代码，只给你一张"按行号逐块带读"的地图。看完以后你可以按自己节奏定位到任意一块去读细节。

---

## 阅读前提

推荐先读：

1. [simple.md](./simple.md)
2. [architecture.md](./architecture.md)

再来看这篇。因为这篇默认你已经知道 Hermes 的高层结构，只差一张“如何走读这个大文件”的地图。

如果你今天只是修一个局部 bug，不需要把本文从头读到尾。按目录跳到对应分区即可。

---

## 一页读懂：这个文件在干什么？

用一句话说清楚：

> **一个循环**：问 LLM → LLM 说"请调用 X 工具" → 调工具 → 把结果塞回去 → 再问 LLM → ... → LLM 说"我做完了" → 返回最终答案。

整个文件的体量之所以爆炸，是因为它同时承担了 **8 件事**：

| 职责 | 你能在文件哪里找到 |
|---|---|
| 1️⃣ 对话主循环 | `run_conversation()` (L7674) |
| 2️⃣ 多厂商 API 适配（OpenAI/Anthropic/Codex/Qwen...） | L3419-6060 大片区域 |
| 3️⃣ 流式 vs 非流式调用 | `_interruptible_api_call` (L4734)、`_interruptible_streaming_api_call` (L4972) |
| 4️⃣ 工具派发（并行/串行） | `_execute_tool_calls_*` (L6833-7483) |
| 5️⃣ 系统提示词构建 | `_build_system_prompt` (L3092) |
| 6️⃣ 上下文压缩 | `_compress_context` (L6719) |
| 7️⃣ 错误恢复/凭证轮换/回退模型 | L4498-5842 |
| 8️⃣ 会话持久化 + 轨迹记录 | L2241-2554 |

---

## 文件分区地图（按行号分块，每块一句话）

### 🔹 Part 0 — 模块级工具函数（L1-513）

这些是类外的小工具，**初读可跳过**，遇到再回来看。

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L1-22 | 模块 docstring | 告诉你这个文件是干嘛的 |
| L23-109 | imports | 依赖一览，看得出它用到了 `agent/`、`tools/`、`hermes_cli/` 等几乎所有子模块 |
| L113-167 | `_SafeWriter` / `_install_safe_stdio` | 包裹 stdout/stderr，防止 print 在断开的管道上崩溃 |
| L170-211 | `IterationBudget` | 线程安全计数器，防止工具调用陷入死循环 |
| L256-344 | `_is_destructive_command` / `_should_parallelize_tool_batch` / `_paths_overlap` | 判断一批工具调用能不能**并行**（读文件能并行，写同一个目录就不行） |
| L345-511 | `_sanitize_*` 系列 | 各种消息体里非法字符的清洗（从 Word 粘贴来的奇怪字符、非 ASCII 等） |
| L513 | `_qwen_portal_headers` | Qwen 厂商专用请求头 |

---

### 🔷 Part 1 — `AIAgent.__init__` & 重置（L526-1639）

**这是 Agent 的"出生仪式"。读这里你会看到 Agent 有哪些状态。**

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L526-540 | 类定义 + 类级常量 | `_context_pressure_last_warned`：跨实例去重的警告冷却表 |
| L541-548 | `base_url` property | 存 API 地址，顺便小写缓存一份方便判断 |
| **L550-1472** | **`__init__`（922 行）** | 读所有配置、建 OpenAI 客户端、装工具集、初始化几十个状态变量。**第一次读可只扫注释** |
| L1473-1511 | `reset_session_state` | 新会话来了，把各种"上一轮残留"清零 |
| L1512-1639 | `switch_model` | 运行中热切换模型（/model 命令触发） |

---

### 🔷 Part 2 — 打印、状态、runtime 检测（L1640-2004）

纯工具方法，帮 Agent 知道"自己现在在什么环境里"。

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L1640-1684 | `_safe_print` / `_vprint` | 日志输出，支持安静模式/详细模式 |
| L1685-1734 | `_should_start_quiet_spinner` 等 | 决定要不要显示转圈圈 |
| L1735-1860 | `_current_main_runtime` / `_check_compression_model_feasibility` | 记录主模型信息、检查压缩模型是否可用 |
| L1861-1895 | `_is_direct_openai_url` / `_is_openrouter_url` / `_model_requires_responses_api` / `_max_tokens_param` | 按 URL/模型名识别"这是哪家 API" |
| L1896-2004 | `_strip_think_blocks` / `_looks_like_codex_intermediate_ack` | 处理模型 reasoning 块的文本（`<think>...</think>`） |

---

### 🔷 Part 3 — 推理提取、任务清理、后台审查（L2005-2240）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L2005-2071 | `_extract_reasoning` | 从响应里抽出 reasoning 文本 |
| L2072-2139 | `_cleanup_task_resources` | 任务结束收尾（关浏览器、关容器等） |
| L2140-2240 | `_spawn_background_review` | 在后台线程里启动一个独立 Agent 做代码审查 |

---

### 🔷 Part 4 — 会话持久化（SQLite）（L2241-2349）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L2241-2258 | `_apply_persist_user_message_override` | 存"干净版"用户消息（API 内部可能加了合成前缀） |
| L2259-2319 | `_persist_session` / `_flush_messages_to_session_db` | 把消息写进 SQLite 会话库 |
| L2320-2349 | `_get_messages_up_to_last_assistant` | 截取到最近一条助手消息（用于断点续存） |

---

### 🔷 Part 5 — 工具格式化 + 轨迹记录（RL 数据）（L2351-2554）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L2351-2374 | `_format_tools_for_system_message` | 把工具列表格式化成文本塞进 system prompt |
| L2375-2539 | `_convert_to_trajectory_format` | 把本轮对话转成训练用的轨迹格式 |
| L2540-2554 | `_save_trajectory` | 写 JSONL 文件 |

---

### 🔷 Part 6 — 错误清洗、日志脱敏（L2556-2791）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L2556-2595 | `_summarize_api_error` | 把冗长的 API 错误简化成一句话 |
| L2596-2628 | `_mask_api_key_for_logs` / `_clean_error_message` | 日志里把 API Key 遮成 `sk-***abc` |
| L2629-2693 | `_extract_api_error_context` | 提取错误里的 request_id、status_code 等 |
| L2694-2791 | `_usage_summary_for_api_request_hook` / `_dump_api_request_debug` | 把完整请求/响应写到调试日志 |

---

### 🔷 Part 7 — 会话 Markdown 日志、中断、速率限制、关闭（L2792-3045）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L2792-2867 | `_clean_session_content` / `_save_session_log` | 把对话保存成人可读的 markdown |
| L2868-2914 | `interrupt` / `clear_interrupt` | 用户按 Ctrl+C 触发这里 |
| L2915-2942 | `_touch_activity` / `_capture_rate_limits` / `get_rate_limit_state` | 记录"上一次活动时间"和 rate limit |
| L2943-3045 | `get_activity_summary` / `shutdown_memory_provider` / `close` / `_hydrate_todo_store` | 活动汇总、关记忆、全量清理、把 todo 从历史里恢复 |
| L3078-3091 | `is_interrupted` 属性 | 一个只读标志位 |

---

### ⭐ Part 8 — 系统提示词构建（核心 #1）（L3092-3262）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| **L3092-3262** | **`_build_system_prompt`** | 把"你是谁 + 你的工具 + 你的技能 + 用户的 SOUL.md + 记忆"拼成一个超长字符串发给 LLM |

📍 **推荐精读** — 理解 Agent "人格"是怎么装配出来的。

---

### 🔷 Part 9 — 消息清洗 + 工具调用修复（L3263-3418）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L3263-3272 | `_get_tool_call_id_static` | 从 tool_call 对象里拿 id |
| L3273-3342 | `_sanitize_api_messages` | 消息历史里的各种结构异常兜底修复 |
| L3343-3390 | `_cap_delegate_task_calls` / `_deduplicate_tool_calls` | 限制 delegate 调用数量、去重 |
| L3391-3429 | `_repair_tool_call` / `_invalidate_system_prompt` | LLM 给了错工具名，尝试映射到正确的 |

---

### 🔷 Part 10 — Codex / Responses API 适配（L3419-4028）

这一大片都是为了支持 OpenAI 新的 **Responses API**（与旧的 Chat Completions API 格式不同）。
**初读可整块跳过**。

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L3430-3480 | `_responses_tools` / `_deterministic_call_id` / `_split_responses_tool_id` | 工具格式/ID 转换 |
| L3481-3612 | `_derive_responses_function_call_id` / `_chat_messages_to_responses_input` | 把 Chat 格式消息转成 Responses 格式 |
| L3614-3832 | `_preflight_codex_input_items` / `_preflight_codex_api_kwargs` | 发送前做一轮合法性体检 |
| L3833-3864 | `_extract_responses_message_text` / `_extract_responses_reasoning_text` | 从响应里抽出文本/推理 |
| L3865-4028 | `_normalize_codex_response` | 把 Responses 格式响应转回 Chat 格式 |

---

### 🔷 Part 11 — OpenAI 客户端生命周期（线程安全）（L4029-4298）

基础设施代码，**初读可跳过**。保证在多线程/断连/socket 泄漏场景下客户端可以被安全重建。

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L4029-4078 | `_thread_identity` / `_client_log_context` / `_openai_client_lock` / `_is_openai_client_closed` | 线程标识 + 锁 + 判断客户端是否已关 |
| L4079-4182 | `_create_openai_client` / `_force_close_tcp_sockets` / `_close_openai_client` | 建/关客户端，强制回收 TCP socket |
| L4183-4285 | `_replace_primary_openai_client` / `_ensure_primary_openai_client` / `_cleanup_dead_connections` | 主客户端的创建、替换、清理僵尸连接 |
| L4286-4298 | `_create_request_openai_client` / `_close_request_openai_client` | 单次请求用的独立客户端 |

---

### 🔷 Part 12 — Codex 流式 + 凭证刷新（L4299-4728）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L4299-4497 | `_run_codex_stream` / `_run_codex_create_stream_fallback` | Responses API 的流式调用 |
| L4498-4600 | `_try_refresh_codex_client_credentials` / `_try_refresh_nous_client_credentials` / `_try_refresh_anthropic_client_credentials` | Token 过期自动换新 |
| L4601-4644 | `_apply_client_headers_for_base_url` / `_swap_credential` | 按地址贴特定 header、切换凭证 |
| L4645-4728 | `_recover_with_credential_pool` | 一组凭证轮流试，哪把能用用哪把 |

---

### ⭐ Part 13 — 非流式 API 调用（核心 #2）（L4729-4877）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L4729-4733 | `_anthropic_messages_create` | Anthropic Messages API 薄包装 |
| **L4734-4877** | **`_interruptible_api_call`** | 最底层的"一次非流式 API 调用"，可被 Ctrl+C 中断 |

---

### ⭐ Part 14 — 流式 API 调用（核心 #3）（L4878-5545）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L4878-4970 | `_reset_stream_delivery_tracking` / `_fire_stream_delta` / `_fire_reasoning_delta` / `_fire_tool_gen_started` | 把流式片段转发给回调（CLI、TTS 等） |
| **L4972-5545** | **`_interruptible_streaming_api_call`（574 行）** | 流式接收文本 delta、工具调用 delta，边收边触发回调 |

---

### 🔷 Part 15 — 回退与恢复（L5546-5842）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L5546-5690 | `_try_activate_fallback` | 主模型挂了，自动切备用模型 |
| L5691-5763 | `_restore_primary_runtime` | 下一轮开始时，恢复主模型 |
| L5764-5842 | `_try_recover_primary_transport` | 主模型传输层（http 连接）恢复尝试 |

---

### 🔷 Part 16 — 视觉/图片 + Qwen 特例（L5843-6060）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L5843-5872 | `_content_has_image_parts` / `_materialize_data_url_for_vision` | 判断消息里是否带图、图片转成 data URL |
| L5873-5983 | `_describe_image_for_anthropic_fallback` / `_preprocess_anthropic_content` / `_prepare_anthropic_messages_for_api` / `_anthropic_preserve_dots` | Anthropic 专用的消息预处理 |
| L5996-6060 | `_is_qwen_portal` / `_qwen_prepare_chat_messages` | Qwen 门户的消息调整 |

---

### ⭐ Part 17 — API 请求参数构建（核心 #4）（L6061-6398）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| **L6061-6332** | **`_build_api_kwargs`（272 行）** | 把一整个 API 请求 body 拼出来：model、messages、tools、temperature、reasoning、max_tokens... |
| L6333-6398 | `_supports_reasoning_extra_body` / `_github_models_reasoning_extra_body` | 判断本模型是否支持 reasoning 参数 |

---

### 🔷 Part 18 — 组装助手消息 + 工具调用清洗（L6399-6557）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L6399-6517 | `_build_assistant_message` | 把 LLM 返回的 response 对象转成我们内部消息格式 |
| L6518-6557 | `_sanitize_tool_calls_for_strict_api` / `_should_sanitize_tool_calls` | 某些严格 API 不接受额外字段，清洗掉 |

---

### ⭐ Part 19 — 记忆 + 上下文压缩（L6558-6832）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L6558-6718 | `flush_memories` | 会话末尾把这轮学到的东西写进持久化记忆 |
| **L6719-6832** | **`_compress_context`** | 对话太长时智能摘要中段，保留首尾 |

---

### ⭐ Part 20 — 工具执行（核心 #5）（L6833-7483）

读懂这里，就理解了"Agent 怎么调工具"。

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L6833-6855 | `_execute_tool_calls` | 入口，选择走并行还是串行 |
| L6856-6929 | `_invoke_tool` | 实际调用一个工具（通过 `model_tools.handle_function_call`） |
| L6930-7135 | `_execute_tool_calls_concurrent` | 并行执行（多线程池） |
| **L7136-7483** | **`_execute_tool_calls_sequential`** | 串行执行（一个一个来） |

---

### 🔷 Part 21 — 压力感知 + 迭代上限（L7484-7673）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L7484-7522 | `_emit_context_pressure` | 给用户发"上下文快满了"的提示 |
| L7523-7673 | `_handle_max_iterations` | 循环达到上限时，让 LLM 总结收尾 |

---

### ⭐⭐ Part 22 — 主循环（核心 #6）（L7674-10569）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| **L7674-10569** | **`run_conversation`（2,895 行！）** | 所有上述模块在这里串成一个完整循环。**这是整个文件最重要的方法。** |
| L10570-10583 | `chat` | `run_conversation` 的一句话简化入口 |

---

### 🔷 Part 23 — CLI 入口（L10585-end）

| 行号 | 名字 | 一句话解释 |
|---|---|---|
| L10585-10796 | `main()` | `python run_agent.py --query="..."` 这样跑时的入口，解析命令行参数并跑一次对话 |
| L10799-end | `if __name__ == "__main__"` | 用 `fire` 库把 `main()` 暴露为 CLI |

---

## 🧭 推荐阅读路径（小白专用）

**不要从第一行开始读。** 按下面的顺序，你 2-3 小时能吃透骨架。

### 第一站：5 分钟，看见全貌
1. 📖 L1-22（模块 docstring） — 确认"这个文件是个 Agent 循环"
2. 📖 L10570-10583（`chat` 方法） — 最简入口，一行调 `run_conversation`
3. 📖 L10585-10796（`main` 函数） — CLI 是怎么用这个类的

### 第二站：30 分钟，理解单次对话的骨架
4. 📖 L170-211（`IterationBudget`） — 热身：理解"迭代预算"概念
5. 📖 L7674-7800（`run_conversation` 前 130 行） — 只看开头的初始化，别钻 for 循环
6. 📖 L3092-3262（`_build_system_prompt`） — Agent 的"人格"怎么装配
7. 📖 L6856-6929（`_invoke_tool`） — 一个工具是怎么被调的
8. 📖 L7136-7300（`_execute_tool_calls_sequential` 前段） — 串行工具执行的骨架

### 第三站：1 小时，看懂主循环
9. 📖 L7800-8500（`run_conversation` 中段） — 主循环的 while + API call
10. 📖 L6061-6200（`_build_api_kwargs` 前段） — 一次 API 请求长什么样
11. 📖 L4734-4877（`_interruptible_api_call`） — 实际的 API 调用

### 第四站（选修）：遇到问题再回来看
- 🔧 L3419-4028（Codex 适配） — 如果你用 GPT-5 / Codex 模型
- 🔧 L4029-4298（客户端生命周期） — 如果你遇到"连接断了"的 bug
- 🔧 L4498-4728（凭证刷新/池化） — 如果你遇到"Token 过期"
- 🔧 L5546-5842（回退机制） — 如果你好奇"主模型挂了怎么办"
- 🔧 L6719-6832（上下文压缩） — 如果你想理解长对话怎么不爆

---

## 💡 给小白的几个忠告

1. **不要试图一次读完**。这个文件是十几年工业场景打磨出来的"补丁合集"，很多代码是特定厂商、特定 bug 的 workaround。
2. **先读主干，再读旁路**。主干 = Part 8/13/14/17/19/20/22。旁路 = 厂商适配、凭证、回退、视觉。
3. **读到陌生的 `_xxx` 方法时**，别急着跳进去。先看方法名猜意思 + 读它的 docstring + 看调用点。读得太深你会迷路。
4. **配合 `readme/architecture.md` 一起读**，那份文档讲了**为什么**这么设计，这份讲**在哪里**。

---

## 📌 如果以后想真的物理拆分

建议按 **"核心 #1 ~ #6"** 为边界抽出 6 个子模块（放到 `agent/` 下）：

| 建议文件 | 抽出的方法 | 当前行号 |
|---|---|---|
| `agent/system_prompt.py` | `_build_system_prompt` | L3092-3262 |
| `agent/api_client.py` | `_interruptible_api_call` + OpenAI 客户端管理 | L4029-4877 |
| `agent/api_streaming.py` | `_interruptible_streaming_api_call` + 流式 delta 派发 | L4878-5545 |
| `agent/api_kwargs.py` | `_build_api_kwargs` | L6061-6398 |
| `agent/tool_dispatch.py` | `_execute_tool_calls_*` + `_invoke_tool` | L6833-7483 |
| `agent/codex_responses.py` | 整个 Codex 适配层 | L3419-4028 |

主类 `AIAgent` 瘦身后只保留 `__init__` + `run_conversation` 的编排骨架，约 3000 行。

但这是另一个大工程，要动全项目 import，得等你对整个循环完全熟了再做。
