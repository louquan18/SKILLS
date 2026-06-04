# Claude Skills 通用技能库

## 概述

这是一套**通用的 Claude Code Skills 集合**，可被任何项目复用。

Skills 是 Claude Code 的领域知识模块，通过 `/skill-name` 调用，让 Claude 具备特定领域的专业能力。

## 可用 Skills

### 通用开发 Skills

| Skill | 调用命令 | 说明 |
|-------|----------|------|
| **System Architect** | `/system-architect` | 系统架构设计、技术选型、ADR 决策记录 |
| **Project Planner** | `/project-planner` | 项目规划、里程碑拆解、Issue 管理 |
| **Python Backend** | `/python-backend` | Python 后端开发（FastAPI、Repository 模式） |
| **Test Engineer** | `/test-engineer` | 测试工程（单元测试、集成测试） |
| **Reviewer** | `/reviewer` | 代码审查（Review Checklist） |
| **Refactor Engineer** | `/refactor-engineer` | 重构工程（Refactor Checklist） |

### Agent/RAG 专项 Skills

| Skill | 调用命令 | 说明 |
|-------|----------|------|
| **Agent Workflow Architect** | `/agent-workflow-architect` | Agent 工作流设计（State Schema、Workflow、Checkpoint） |
| **RAG & Memory Engineer** | `/rag-memory-engineer` | RAG 系统、上下文管理、向量检索、记忆压缩 |

### 领域示例 Skills

| Skill | 调用命令 | 说明 |
|-------|----------|------|
| **Novel Skill Engineer** | `/novel-skill-engineer` | [示例] 小说领域特定 skill，可参考创建自己的领域 skill |

## 在其他项目中使用

### 方式 1: Git Submodule（推荐）

```bash
# 在目标项目中添加为 submodule
git submodule add <this-repo-url> .claude/skills

# 更新到最新版本
cd .claude/skills && git pull
```

### 方式 2: 直接复制

```bash
# 复制 skills 目录到目标项目
cp -r .claude/skills/ /path/to/your/project/.claude/skills/
```

### 方式 3: 符号链接

```bash
# 创建符号链接
ln -s /path/to/this/repo/.claude/skills /path/to/your/project/.claude/skills
```

## 目录结构

```
.claude/
├── CLAUDE.md           # 本文件
├── settings.json       # 权限配置
└── skills/             # Skills 目录
    ├── system-architect/
    │   └── skill.md
    ├── project-planner/
    │   ├── skill.md
    │   ├── milestone-template.md
    │   └── issue-template.md
    ├── python-backend/
    │   ├── skill.md
    │   ├── fastapi-template.md
    │   └── repository-template.md
    ├── agent-workflow-architect/
    │   ├── skill.md
    │   ├── state-schema-template.md
    │   ├── workflow-template.md
    │   └── checkpoint-template.md
    ├── rag-memory-engineer/
    │   └── skill.md
    ├── test-engineer/
    │   └── skill.md
    ├── reviewer/
    │   ├── skill.md
    │   └── review-checklist.md
    ├── refactor-engineer/
    │   └── skill.md
    └── novel-skill-engineer/    # 领域示例
        └── skill.md
```

## Skill 编写规范

每个 skill 是一个目录，包含：

- **skill.md**（必需）- Skill 定义文件，包含 frontmatter 和完整说明
- **模板文件**（可选）- 相关模板和参考文档

### skill.md 格式

```markdown
---
skill: skill-name
description: 一句话说明
tags: [tag1, tag2, tag3]
---

# Skill Name

## 职责范围
...

## 工作流程
...

## 最佳实践
...

## 相关 Skills
...
```

## 开发建议

1. **按阶段使用** - 不同开发阶段使用不同 skill
2. **组合使用** - Skills 之间可以相互引用
3. **自定义扩展** - 基于现有 skill 创建项目专属 skill
4. **领域适配** - 参考 `novel-skill-engineer` 创建领域特定 skill

## 许可

自由使用和修改。
