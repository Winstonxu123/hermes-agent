# Skills 技能系统指南

技能（Skills）是 Hermes Agent 最强大的扩展机制。每个技能是一份 Markdown 文档，描述了智能体在特定场景下应该如何行为、使用什么工具、遵循什么步骤。

---

## 技能的本质

技能本质上是一份**增强提示词**，当用户的请求匹配到某个技能时，该技能的内容会被注入到系统提示词中，指导智能体的行为。

每个技能是一个目录，核心文件是 `SKILL.md`：

```
skills/
└── github/
    └── code-review/
        ├── SKILL.md          # 技能描述和指令
        └── resources/        # 可选的附加资源
```

---

## 技能目录结构

### 内置技能（`skills/`）

安装时自动复制到 `~/.hermes/skills/`，约 30+ 类别：

| 类别 | 目录 | 示例技能 |
|------|------|----------|
| Apple 生态 | `apple/` | iMessage、Reminders、Notes、Find My |
| 自主智能体 | `autonomous-ai-agents/` | Claude Code、Codex、OpenCode 集成 |
| 创意工具 | `creative/` | ASCII 艺术、视频、p5.js、Manim、Excalidraw |
| 数据科学 | `data-science/` | Jupyter 实时内核 |
| DevOps | `devops/` | Webhook 订阅 |
| 图表制作 | `diagramming/` | 各种图表生成 |
| GitHub | `github/` | 认证、Issue、PR、代码审查 |
| 媒体工具 | `media/` | YouTube、Songsee、音乐 |
| ML 工具 | `mlops/` | HuggingFace、向量数据库、训练、评估 |
| 笔记工具 | `note-taking/` | Obsidian 集成 |
| 生产力 | `productivity/` | Google Workspace、Linear、Notion |
| 研究 | `research/` | Arxiv、博客监控、LLM Wiki |
| 社交媒体 | `social-media/` | Twitter/X |
| 软件开发 | `software-development/` | 规划、调试、测试、代码审查 |
| 智能家居 | `smart-home/` | 灯光控制等 |

### 可选技能（`optional-skills/`）

不默认安装，通过 Hub 发现和安装：

```bash
hermes skills          # 浏览技能 Hub
```

包含：区块链工具、Blender MCP、健康 BCI、更多 ML 工具等。

---

## SKILL.md 格式

每个技能的核心是 `SKILL.md` 文件，包含 YAML Frontmatter 和 Markdown 正文：

```markdown
---
platforms: [macos, linux]       # 可选：限制支持的操作系统
tags: [automation, python]      # 可选：分类标签
requires: [github_token]        # 可选：所需的环境变量
---

# 技能名称

技能的简短描述（第一段会被提取为摘要）。

## 使用场景

描述什么时候应该触发这个技能...

## 操作步骤

1. 第一步...
2. 第二步...

## 注意事项

- 重要提醒...
```

### Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `platforms` | `list[str]` | 支持的操作系统：`macos`、`linux`、`windows` |
| `tags` | `list[str]` | 分类标签 |
| `requires` | `list[str]` | 所需环境变量 |

---

## 技能的生命周期

### 1. 发现

在系统提示词构建时，`agent/prompt_builder.py` 调用 `build_skills_system_prompt()` 扫描所有技能目录，生成索引。

### 2. 匹配

当用户输入匹配到某个技能的描述或标签时，智能体会选择使用对应的技能工具：

```
用户: "帮我做一个 code review"
        │
        ▼
智能体识别 → 使用 skills_tool 查看 github/code-review 技能
        │
        ▼
技能内容注入上下文 → 按照技能指令执行
```

### 3. 执行

智能体按照技能中定义的步骤，使用相应的工具完成任务。

### 4. 创建与改进

智能体可以自动将复杂任务提炼为新技能，或改进现有技能：

```
用户: "把刚才做的事情创建成一个技能"
        │
        ▼
skill_manager_tool.create_skill()
        │
        ▼
新技能保存到 ~/.hermes/skills/
```

---

## 管理技能

### 查看可用技能

```bash
hermes skills                # 交互式浏览
```

在聊天中：

```
/skills                      # 列出所有技能
```

### 禁用技能

在 `~/.hermes/config.yaml` 中：

```yaml
skills:
  disabled:
    - social-media/xitter    # 禁用特定技能
```

### 添加外部技能目录

```yaml
skills:
  external_dirs:
    - /path/to/my/skills     # 添加自定义技能目录
```

---

## 创建自己的技能

### 手动创建

1. 在 `~/.hermes/skills/` 下创建目录：

```bash
mkdir -p ~/.hermes/skills/my-category/my-skill
```

2. 创建 `SKILL.md`：

```markdown
---
tags: [automation]
---

# 我的技能

这个技能帮助用户完成某个特定任务。

## 步骤

1. 首先...
2. 然后...
3. 最后...
```

3. 重启 Hermes，新技能会自动被发现。

### 让智能体帮你创建

在对话中：

```
你: 帮我创建一个技能，用于自动化部署到 Vercel
智能体: （使用 create_skill 工具，生成 SKILL.md）
```

### 技能编写最佳实践

- **描述要具体** — 第一段是索引摘要，要清晰说明技能的用途
- **步骤要明确** — 用编号列表写清具体操作步骤
- **包含示例** — 给出输入输出示例，让智能体更好理解
- **声明依赖** — 在 frontmatter 中声明所需的环境变量和平台
- **保持聚焦** — 一个技能解决一个具体问题，不要做太通用

---

## 技能系统的关键代码

| 文件 | 说明 |
|------|------|
| `agent/skill_utils.py` | 技能元数据解析（frontmatter、平台匹配） |
| `agent/skill_commands.py` | `/skill-name` 命令解析 |
| `agent/prompt_builder.py` | 技能索引构建（`build_skills_system_prompt()`） |
| `tools/skills_tool.py` | `view_skill`、`list_skills` 工具 |
| `tools/skill_manager_tool.py` | `create_skill`、`update_skill` 工具 |
| `hermes_cli/skills_config.py` | CLI 技能配置 |
| `hermes_cli/skills_hub.py` | 技能 Hub 浏览与安装 |
