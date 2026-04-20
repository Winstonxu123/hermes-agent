# Skills 技能系统指南

Skills 是 Hermes 的“可复用操作手册”。它们不是 Python 工具，而是用 Markdown 写的执行指导，帮助 agent 在遇到某类任务时采用更稳定的流程。

---

## 先分清楚 3 个概念

| 类型 | 本质 | 位置 | 作用 |
|------|------|------|------|
| Skill | Markdown 指南 | `skills/` 或 `~/.hermes/skills/` | 告诉模型“这类任务该怎么做” |
| Tool | Python 能力 | `tools/` | 给模型可调用的能力 |
| Plugin | 运行时扩展机制 | `plugins/`、`~/.hermes/plugins/` | 扩展 tool、hook、provider 等 |

一句话记忆：

- Tool 决定“能做什么”
- Skill 决定“应该怎么做”
- Plugin 决定“怎么接新的运行时扩展”

---

## Skills 的本质

Skill 更像：

- 给 agent 的 SOP
- 面向特定任务域的操作指南
- 一段可检索、可注入提示词的经验

每个 Skill 通常是一个目录，核心文件是 `SKILL.md`：

```text
skills/
└── github/
    └── code-review/
        ├── SKILL.md
        └── resources/
```

---

## 技能目录

### 内置技能

仓库里的 `skills/` 会在安装后复制到 `~/.hermes/skills/`。这些是官方或内置技能。

### 可选技能

`optional-skills/` 里的内容不会默认安装，通常通过：

```bash
hermes skills
```

来浏览和安装。

---

## 一个 `SKILL.md` 长什么样

```markdown
---
platforms: [macos, linux]
tags: [automation, python]
requires: [GITHUB_TOKEN]
---

# 技能名称

这里的第一段最好就是一句清晰摘要，因为它会影响索引和触发效果。

## 使用场景

说明什么时候该用这个 skill。

## 步骤

1. 先做什么
2. 再做什么
3. 最后做什么

## 注意事项

- 关键限制
- 风险点
- 常见失败原因
```

### Frontmatter 字段

| 字段 | 说明 |
|------|------|
| `platforms` | 限制支持的平台或系统 |
| `tags` | 帮助检索和触发 |
| `requires` | 依赖的环境变量或外部条件 |

---

## Skills 的生命周期

### 1. 被发现

在 system prompt 构建期间，Hermes 会扫描技能目录并生成技能索引。

### 2. 被匹配

当用户任务和 skill 描述、标签或内容足够接近时，agent 会选择查看或使用它。

### 3. 被执行

Skill 本身不执行代码，但会指导 agent 使用合适的工具和步骤完成任务。

### 4. 被沉淀和迭代

复杂任务可以被总结成新 skill，现有 skill 也可以持续更新。

---

## 新人最常见的两个场景

### 场景 1：我想先用现成技能

1. 运行 `hermes skills`
2. 看看有哪些分类和技能
3. 在会话里提出一个比较明确的任务
4. 必要时让 agent 查看具体 skill

### 场景 2：我想沉淀自己的 SOP

1. 手写一个最小 `SKILL.md`
2. 放到 `~/.hermes/skills/<category>/<name>/`
3. 重启 Hermes
4. 试着用自然语言触发它
5. 根据执行结果继续改

---

## 管理技能

### 查看可用技能

```bash
hermes skills
```

在聊天里也可以：

```text
/skills
```

### 禁用技能

```yaml
skills:
  disabled:
    - social-media/xitter
```

### 添加外部技能目录

```yaml
skills:
  external_dirs:
    - /path/to/my/skills
```

这对团队很有用，因为你可以把技能目录放在单独仓库里共同维护。

---

## 创建自己的技能

### 手动创建

```bash
mkdir -p ~/.hermes/skills/my-category/my-skill
```

然后创建 `SKILL.md`。

### 让 Hermes 帮你创建

你也可以在会话里直接要求 agent 帮你生成一个 skill，然后再人工审阅它。

---

## 编写最佳实践

- 第一段摘要要具体，直接说清楚用途
- 步骤要编号，不要写得太散
- 一个 skill 解决一个窄问题
- 适当写失败场景和注意事项
- `requires` 只写真正影响执行的依赖

额外给新人的两条建议：

- 先写小 skill，再写大 skill
- skill 描述流程和策略，不要重复工具 API 说明

---

## 和代码实现对应的关键位置

| 文件 | 说明 |
|------|------|
| `agent/skill_utils.py` | skill 元数据解析 |
| `agent/skill_commands.py` | `/skill-name` 相关命令支持 |
| `agent/prompt_builder.py` | 技能索引和注入 |
| `tools/skills_tool.py` | 查看和列出技能 |
| `tools/skill_manager_tool.py` | 创建和更新技能 |
| `hermes_cli/skills_config.py` | 技能配置 |
| `hermes_cli/skills_hub.py` | 技能浏览和安装 |
