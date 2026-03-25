# Open Claw (Claude Code) 深度使用文档：Skill 系统

## 目录

- [1. 什么是 Skill](#1-什么是-skill)
- [2. 内置 Skill 一览](#2-内置-skill-一览)
- [3. 如何使用 Skill](#3-如何使用-skill)
- [4. 如何创建自定义 Skill](#4-如何创建自定义-skill)
- [5. Skill 文件结构详解](#5-skill-文件结构详解)
- [6. Skill 存储位置与作用域](#6-skill-存储位置与作用域)
- [7. Skill 的高级特性](#7-skill-的高级特性)
- [8. 探索与发现新 Skill](#8-探索与发现新-skill)
- [9. Skill 社区与生态](#9-skill-社区与生态)
- [10. 团队共享 Skill](#10-团队共享-skill)
- [11. 实战案例](#11-实战案例)
- [12. Skill 与其他扩展机制对比](#12-skill-与其他扩展机制对比)
- [13. 编写 Skill 的最佳实践](#13-编写-skill-的最佳实践)
- [14. 参考资源](#14-参考资源)

---

## 1. 什么是 Skill

**Skill（技能）** 是 Open Claw 中可复用的结构化扩展，用 Markdown 文件定义，教会 Claude 如何执行特定任务或应用领域知识。

### 核心特征

- **Markdown + YAML frontmatter** 格式（`.SKILL.md` 文件）
- **两种调用方式：**
  - **用户调用（User-invoked）**：你在交互模式中输入 `/skill-name` 手动触发
  - **模型调用（Model-invoked）**：Claude 根据上下文自动判断何时使用
- 支持**动态上下文注入**、参数传递、工具限制
- 可包含**辅助文件**（模板、示例、脚本）
- 遵循开放的 **Agent Skills 标准**，跨工具兼容

### Skill vs 传统 Commands

```
旧方式（仍兼容）：
  .claude/commands/deploy.md  →  /deploy

新方式（推荐）：
  .claude/skills/deploy/SKILL.md  →  /deploy

新方式的额外能力：
  - 辅助文件（示例、模板、脚本）
  - Frontmatter 控制（模型、推理强度、工具限制）
  - 子 Agent 隔离执行（context: fork）
  - 动态上下文注入（Shell 命令输出）
```

> 如果 Skill 和 Command 同名，**Skill 优先**。

---

## 2. 内置 Skill 一览

Open Claw 自带以下 Skill：

| Skill | 命令 | 功能 |
|-------|------|------|
| **batch** | `/batch <指令>` | 大规模并行修改。生成 5-30 个隔离 Agent，各自在独立 git 工作树中工作 |
| **simplify** | `/simplify [聚焦点]` | 代码质量审查。生成 3 个并行审查 Agent，检查复用性、质量和效率 |
| **loop** | `/loop [间隔] <提示词>` | 定时重复执行。在会话期间按间隔反复运行指定提示词 |
| **schedule** | `/schedule` | 创建定时远程 Agent。设置 cron 定时任务 |
| **claude-api** | `/claude-api` | 加载 Claude API 参考文档。当代码导入 `anthropic` SDK 时自动触发 |
| **debug** | `/debug [描述]` | 调试当前会话。读取调试日志排查问题 |
| **update-config** | `/update-config` | 配置 Claude Code。修改 settings.json、Hook、权限等 |
| **keybindings-help** | `/keybindings` | 自定义快捷键。修改 keybindings.json |

### 使用示例

```bash
# 批量迁移 src/ 下的代码从 Solid 到 React
/batch migrate src/ from Solid to React

# 审查改动的代码质量，聚焦内存效率
/simplify focus on memory efficiency

# 每 5 分钟检查部署是否完成
/loop 5m check if deploy finished

# 加载 Claude API 参考
/claude-api
```

---

## 3. 如何使用 Skill

### 3.1 用户调用的 Skill

在交互模式中输入 `/` 后跟 Skill 名称：

```bash
/skill-name
/skill-name [参数]
/skill-name 可选参数
```

示例：

```bash
/fix-issue 123
/migrate-component SearchBar React Vue
/explain-code src/auth/login.ts
```

### 3.2 模型自动调用的 Skill

无需特殊语法，正常与 Claude 对话即可。Claude 会根据 Skill 的 `description` 字段判断何时使用。

例如，如果有一个描述为 "Generate API documentation with examples" 的 Skill，当你说"给 API 生成文档"时，Claude 会自动调用它。

### 3.3 查看可用 Skill

```bash
/help           # 查看所有可用的 Skill 和命令
/               # 输入斜杠后等待自动补全建议
```

### 3.4 重新加载 Skill

编辑 Skill 文件后，无需重启会话：

```bash
/reload-plugins   # 重新加载所有 Skill 和 Plugin
```

---

## 4. 如何创建自定义 Skill

### 4.1 最小 Skill

创建目录和 `SKILL.md` 文件即可：

```bash
mkdir -p .claude/skills/greet
```

`.claude/skills/greet/SKILL.md`:

```yaml
---
name: greet
description: 用中文打招呼并介绍项目概况
---

用中文向用户打招呼，然后简要介绍当前项目的技术栈和目录结构。
```

现在在交互模式中输入 `/greet` 即可使用。

### 4.2 带参数的 Skill

使用 `$ARGUMENTS` 占位符接收用户输入：

```yaml
---
name: explain-file
description: 详细解释指定文件的代码逻辑
argument-hint: [文件路径]
---

请详细解释 $ARGUMENTS 这个文件：

1. 文件的整体作用
2. 主要的函数/类及其职责
3. 关键的数据流
4. 值得注意的设计模式或技巧
```

使用：`/explain-file src/auth/middleware.ts`

### 4.3 带工具限制的 Skill

```yaml
---
name: safe-audit
description: 只读安全审计，不做任何修改
allowed-tools: Read, Grep, Glob
---

对 $ARGUMENTS 进行安全审计：
- 检查硬编码的密钥
- 验证输入处理
- 审查认证逻辑
- 检查 SQL 注入风险
```

### 4.4 带动态上下文的 Skill

使用 `` !`命令` `` 语法在 Skill 加载时执行 Shell 命令并注入输出：

```yaml
---
name: pr-summary
description: 总结当前 Pull Request
context: fork
---

## PR 上下文
- PR diff: !`gh pr diff`
- PR 评论: !`gh pr view --comments`
- 修改的文件: !`gh pr diff --name-only`

请基于以上信息，用中文写一份 PR 总结：
1. 核心改动
2. 具体代码变更引用
3. 讨论要点
```

### 4.5 在子 Agent 中运行的 Skill

使用 `context: fork` 在隔离环境中执行，避免污染主会话上下文：

```yaml
---
name: deep-analysis
description: 深度代码分析（在隔离 Agent 中运行）
context: fork
agent: Explore
---

对 $ARGUMENTS 进行深度分析：
1. 查找所有相关的安全文件
2. 分析认证、授权、输入验证
3. 汇总发现并按严重程度排序
```

---

## 5. Skill 文件结构详解

### 5.1 单文件 Skill

最简形式：

```
my-skill/
└── SKILL.md       ← 必需，包含指令和 frontmatter
```

### 5.2 带辅助文件的复杂 Skill

```
doc-generator/
├── SKILL.md           ← 必需，主要指令
├── reference.md       ← 详细 API 文档
├── examples.md        ← 使用示例
├── templates/
│   ├── api-endpoint.md
│   └── function.md
└── scripts/
    └── validate.sh    ← 辅助脚本
```

在 SKILL.md 中引用辅助文件：

```markdown
---
name: doc-generator
description: 生成 API 文档和示例
---

# API 文档生成器

详细用法见 [examples.md](examples.md)。
完整配置选项见 [reference.md](reference.md)。

## 步骤
1. 读取并理解代码
2. 按 [templates/](templates/) 中的模板生成文档
3. 使用 [validate.sh](scripts/validate.sh) 验证
```

> 建议 SKILL.md 保持 **500 行以内**，详细内容移到辅助文件。

### 5.3 Frontmatter 字段参考

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 否 | 显示名称（小写、连字符/数字，最长 64 字符）。默认使用目录名 |
| `description` | 推荐 | 何时使用此 Skill。Claude 据此判断是否自动调用 |
| `argument-hint` | 否 | 参数提示，显示在自动补全中。如 `[issue-number]`、`[filename] [format]` |
| `disable-model-invocation` | 否 | 设为 `true` 禁止 Claude 自动调用。适用于有副作用的工作流 |
| `user-invocable` | 否 | 设为 `false` 隐藏于 `/` 菜单。用于背景知识型 Skill |
| `allowed-tools` | 否 | 限制可用工具：`Read, Grep, Glob, Bash, Edit, Write` |
| `model` | 否 | 覆盖模型：`sonnet`、`opus`、`haiku` 或完整模型 ID |
| `effort` | 否 | 覆盖推理强度：`low`、`medium`、`high`、`max` |
| `context` | 否 | 设为 `fork` 在隔离子 Agent 中运行 |
| `agent` | 否 | 与 `context: fork` 配合，指定子 Agent 类型 |
| `hooks` | 否 | 仅在此 Skill 激活时生效的 Hook |

---

## 6. Skill 存储位置与作用域

### 6.1 作用域层级

当同名 Skill 存在于多个位置时，高优先级的覆盖低优先级的：

| 位置 | 作用域 | 优先级 | 共享方式 |
|------|--------|--------|----------|
| 企业管理设置 | 全组织 | 1（最高） | 管理员控制台下发 |
| `~/.claude/skills/<name>/SKILL.md` | 个人所有项目 | 2 | 不共享（仅本人） |
| `.claude/skills/<name>/SKILL.md` | 当前项目 | 3 | 提交到 git（团队共享） |
| Plugin 的 `skills/<name>/SKILL.md` | Plugin 启用的项目 | 4（最低） | 随 Plugin 分发 |

### 6.2 自动发现

- `.claude/skills/` 目录下的 Skill 自动发现
- 支持嵌套子目录（适用于 Monorepo 中按 package 定义 Skill）
- 通过 `--add-dir` 添加的目录中的 Skill 也会自动加载并实时热更新

### 6.3 推荐的目录组织

```
.claude/
├── skills/
│   ├── code-review/
│   │   └── SKILL.md
│   ├── doc-generator/
│   │   ├── SKILL.md
│   │   ├── templates/
│   │   └── examples.md
│   ├── deploy/
│   │   └── SKILL.md
│   └── test-generator/
│       └── SKILL.md
├── agents/
│   ├── reviewer.md
│   └── deployer.md
├── rules/
│   ├── api-rules.md
│   └── frontend-rules.md
└── settings.json
```

---

## 7. Skill 的高级特性

### 7.1 Skill 内定义 Hook

Skill 可以定义仅在自身激活时生效的 Hook：

```yaml
---
name: auto-formatter
description: 编辑文件后自动格式化
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "prettier --write $FILE_PATH"
---

编辑代码后自动使用 Prettier 格式化。
```

### 7.2 Shell 命令动态注入上下文

`` !`命令` `` 语法在 Skill 加载时执行命令，将输出注入 Claude 的上下文：

```yaml
---
name: status-report
description: 生成项目状态报告
---

## 当前状态
- Git 状态: !`git status --short`
- 最近提交: !`git log --oneline -5`
- 分支列表: !`git branch -a`
- 依赖检查: !`npm outdated 2>/dev/null || echo "无过期依赖"`

基于以上信息，生成一份项目状态报告。
```

### 7.3 在子 Agent 中隔离运行

```yaml
---
name: codebase-analyzer
description: 全代码库分析（约消耗 10k token，建议在新会话中使用）
context: fork
agent: Explore
allowed-tools: Read, Grep, Glob, Bash
---

扫描整个代码库：
1. 分析目录结构和模块划分
2. 识别技术栈和框架
3. 找出代码热点（频繁修改的文件）
4. 评估代码质量指标
5. 生成架构概览图
```

`context: fork` 的好处：
- 大量搜索结果留在子 Agent 内，不污染主会话
- 子 Agent 有独立的上下文窗口
- 主会话只收到摘要结果

### 7.4 Skill 中使用 $ARGUMENTS

`$ARGUMENTS` 会被替换为用户在 `/skill-name` 后面输入的所有文本：

```yaml
---
name: review-pr
description: 审查指定的 Pull Request
argument-hint: [PR 编号]
---

审查 PR #$ARGUMENTS：
1. 获取 PR 的 diff
2. 检查代码质量
3. 查找潜在 Bug
4. 评估测试覆盖
5. 给出审查意见
```

使用：`/review-pr 456`

---

## 8. 探索与发现新 Skill

### 8.1 在 Open Claw 内部探索

```bash
# 查看所有可用 Skill 和命令
/help

# 输入 / 等待自动补全
/

# 浏览官方 Plugin 市场
/plugin

# 查看可用的子 Agent（与 Skill 相关）
/agents

# 重新加载 Skill（编辑后）
/reload-plugins
```

### 8.2 自然语言探索

直接问 Claude：

```
有哪些可用的 Skill？
有没有用于代码审查的 Skill？
列出所有项目级别的 Skill
```

### 8.3 创建"Skill 发现器"

你可以创建一个元 Skill 来列出所有可用 Skill：

```yaml
---
name: discover
description: 列出所有可用的 Skill 及其用途
disable-model-invocation: true
---

# 可用 Skill 一览

请列出当前项目中所有可用的 Skill，包括：
1. Skill 名称和调用方式
2. 功能描述
3. 是否支持参数
4. 使用示例

按类别分组展示（开发、测试、部署、文档等）。
```

---

## 9. Skill 社区与生态

### 9.1 官方 Plugin 市场

Open Claw 有内置的 Plugin 市场，Plugin 可以包含 Skill：

```bash
/plugin          # 浏览官方市场
```

**市场分类：**
- Code Intelligence（代码智能）
- Development Workflows（开发工作流）
- Output Styles（输出风格）
- External Integrations（外部集成）

**提交你的 Plugin：**
- Claude.ai 用户：https://claude.ai/settings/plugins/submit
- Console 用户：https://platform.claude.com/plugins/submit

### 9.2 GitHub 社区

在 GitHub 上搜索以下关键词发现社区 Skill：

- `claude-code-skill`
- `claude-code-plugin`
- `agent-skills`
- `claude-code-commands`

### 9.3 Agent Skills 开放标准

Open Claw 的 Skill 系统实现了 **Agent Skills** 开放标准：

- 官网：https://agentskills.io
- 跨工具兼容：同一个 Skill 可以在不同的 AI 工具中使用
- 社区贡献和标准化

### 9.4 分享你的 Skill

**方式一：GitHub 仓库**

创建一个包含 Skill 的仓库，其他人可以克隆使用：

```
my-awesome-skills/
├── README.md
├── skills/
│   ├── skill-a/
│   │   └── SKILL.md
│   └── skill-b/
│       ├── SKILL.md
│       └── templates/
└── LICENSE
```

**方式二：作为 Plugin 发布**

打包为 Plugin 格式，提交到官方市场或通过 URL 分发。

**方式三：Gist / 代码片段**

对于简单的单文件 Skill，直接分享 SKILL.md 的内容即可。

---

## 10. 团队共享 Skill

### 10.1 方式一：项目级 Skill（推荐）

将 `.claude/skills/` 提交到版本控制：

```bash
git add .claude/skills/
git commit -m "添加团队共享 Skill：代码审查和文档生成"
git push
```

团队成员 pull 后自动获得所有 Skill。

### 10.2 方式二：Plugin 分发

创建一个 Plugin 项目：

```bash
mkdir my-team-skills
mkdir -p my-team-skills/.claude-plugin

cat > my-team-skills/.claude-plugin/plugin.json << 'EOF'
{
  "name": "team-skills",
  "description": "团队共享的开发工作流 Skill",
  "version": "1.0.0"
}
EOF

mkdir -p my-team-skills/skills/code-review
# 添加你的 Skill 文件...
```

团队成员安装：

```bash
/plugin install https://github.com/yourteam/team-skills
```

### 10.3 方式三：企业统一下发

使用 Managed Settings（管理员控制台）将 Skill 部署到全组织所有用户。

---

## 11. 实战案例

### 11.1 案例一：自动更新 API 文档

```yaml
---
name: update-api-docs
description: 从代码注释自动更新 API 文档
disable-model-invocation: true
allowed-tools: Read, Grep, Bash, Edit
argument-hint: [目标目录，默认 src/routes]
---

# 更新 API 文档

1. 查找 ${ARGUMENTS:-src/routes} 中所有端点文件（*.ts）
2. 提取 JSDoc 注释和类型定义
3. 按模板生成 Markdown 文档
4. 更新 docs/api.md
5. 运行 Markdown lint 检查
6. 如果有改动，列出修改摘要
```

使用：

```bash
/update-api-docs
/update-api-docs src/api/v2
```

### 11.2 案例二：智能 Code Review

```yaml
---
name: review
description: 审查代码变更，关注安全性、性能和可维护性
context: fork
allowed-tools: Read, Grep, Glob, Bash
---

## 审查上下文
- 变更文件: !`git diff --name-only HEAD~1`
- 详细 diff: !`git diff HEAD~1`

## 审查清单

请按以下维度审查这些变更：

### 安全性
- [ ] 是否有硬编码密钥
- [ ] 输入是否经过验证
- [ ] 是否有 SQL/XSS 注入风险

### 性能
- [ ] 是否有 N+1 查询
- [ ] 是否有不必要的循环
- [ ] 内存使用是否合理

### 可维护性
- [ ] 命名是否清晰
- [ ] 是否有重复代码
- [ ] 错误处理是否完善

### 测试
- [ ] 新代码是否有测试覆盖
- [ ] 边界条件是否考虑

对每个发现的问题，给出：
1. 问题位置（文件:行号）
2. 严重程度（高/中/低）
3. 修改建议
```

### 11.3 案例三：测试生成器

```yaml
---
name: gen-test
description: 为指定文件生成单元测试
argument-hint: [源文件路径]
allowed-tools: Read, Grep, Glob, Write, Bash
---

# 测试生成器

为 $ARGUMENTS 生成全面的单元测试：

## 步骤
1. 读取源文件，理解所有导出的函数/类
2. 查看项目中已有的测试文件，学习测试风格和框架
3. 为每个函数/方法生成测试用例：
   - 正常输入
   - 边界条件
   - 错误情况
   - 类型检查（如适用）
4. 将测试写入对应的测试文件（遵循项目的测试文件命名规范）
5. 运行测试确保通过
```

使用：`/gen-test src/utils/parser.ts`

### 11.4 案例四：定时监控部署

```bash
# 每 3 分钟检查部署状态
/loop 3m 检查 GitHub Actions 的部署流水线是否完成，如果完成了报告结果

# 每 10 分钟检查 PR 状态
/loop 10m 检查我的待审 PR 列表，如果有新评论就总结
```

### 11.5 案例五：批量迁移

```bash
# 将所有组件从 Class 组件迁移到函数式组件
/batch 将 src/components/ 下的所有 Class 组件改写为函数式组件，使用 hooks

# 统一错误处理方式
/batch 将 src/api/ 下的所有 try-catch 替换为统一的 errorHandler 中间件
```

`/batch` 会自动：
1. 分析受影响的文件
2. 生成 5-30 个独立 Agent
3. 每个 Agent 在隔离的 git 工作树中工作
4. 并行执行修改
5. 汇总结果

---

## 12. Skill 与其他扩展机制对比

| 特性 | Skill | Subagent | Hook | MCP | Plugin |
|------|-------|----------|------|-----|--------|
| **可复用提示词** | 是 | 是 | - | - | - |
| **自动化工作流** | 是 | 是 | 是（强） | - | - |
| **团队共享** | 是 | 是 | 是 | 是 | 是（强） |
| **接入外部工具** | - | - | 是 | 是（强） | - |
| **上下文隔离** | 是 | 是（强） | - | - | - |
| **工具权限限制** | 是 | 是（强） | - | - | - |
| **事件驱动** | - | - | 是（强） | - | - |
| **市场分发** | - | - | - | - | 是（强） |
| **参数化输入** | 是 | - | - | - | - |
| **定时执行** | 是（/loop） | - | - | - | - |

### 何时选择什么

| 需求 | 推荐 |
|------|------|
| 可复用的工作流程（审查、文档生成） | **Skill** |
| 隔离执行大量搜索/分析 | **Subagent**（或 Skill + `context: fork`） |
| 工具调用前后的自动验证 | **Hook** |
| 接入外部 API/数据库 | **MCP** |
| 打包分发整套工具链 | **Plugin**（包含 Skill + Hook + MCP） |

---

## 13. 编写 Skill 的最佳实践

### 13.1 写清晰、具体的 description

description 是 Claude 决定何时自动使用 Skill 的依据：

```yaml
# 不好 - 太模糊
description: 代码工具

# 好 - 具体说明何时使用
description: 生成 REST API 文档和使用示例。用于记录 API 端点、编写 README 章节或解释函数行为时使用。
```

### 13.2 保持 SKILL.md 简洁

- 主文件控制在 **500 行以内**
- 详细参考内容移到辅助文件
- 使用相对链接引用辅助文件

### 13.3 用 allowed-tools 限制权限

不需要修改文件的 Skill，明确限制工具：

```yaml
allowed-tools: Read, Grep, Glob    # 只读
```

### 13.4 为有副作用的 Skill 禁用自动调用

部署、提交等有副作用的操作应禁止自动调用：

```yaml
disable-model-invocation: true
```

### 13.5 善用 argument-hint

好的参数提示帮助用户快速理解用法：

```yaml
argument-hint: [文件路径]
argument-hint: [issue 编号]
argument-hint: [源框架] [目标框架]
```

### 13.6 包含使用示例

在辅助文件或 SKILL.md 末尾提供示例：

```markdown
## 使用示例

- `/my-skill src/auth/` - 审计认证模块
- `/my-skill src/api/v2/users.ts` - 审计特定文件
```

### 13.7 标注上下文开销

对于会读取大量文件的 Skill，在 description 中标注：

```yaml
description: 全代码库分析（约消耗 10k token，建议在新会话中使用）
```

### 13.8 测试和迭代

```bash
# 创建或编辑 Skill 后
/reload-plugins          # 重新加载

# 测试调用
/my-skill test-arg

# 确认出现在帮助中
/help
```

---

## 14. 参考资源

### 官方文档

| 主题 | 链接 |
|------|------|
| Skill 系统 | https://code.claude.com/docs/en/skills |
| Hook 指南 | https://code.claude.com/docs/en/hooks-guide |
| 子 Agent | https://code.claude.com/docs/en/sub-agents |
| Plugin 系统 | https://code.claude.com/docs/en/plugins |
| MCP 集成 | https://code.claude.com/docs/en/mcp |
| 最佳实践 | https://code.claude.com/docs/en/best-practices |
| Agent SDK | https://platform.claude.com/docs/agents |

### 社区资源

| 资源 | 说明 |
|------|------|
| Agent Skills 标准 | https://agentskills.io |
| Plugin 市场 | Open Claw 内运行 `/plugin` |
| GitHub 搜索 | 搜索 `claude-code-skill`、`claude-code-plugin` |
| Claude Code Issues | https://github.com/anthropics/claude-code/issues |

### 快速参考：Skill 文件模板

```yaml
---
name: my-skill
description: 一句话描述何时使用此 Skill
argument-hint: [参数说明]
disable-model-invocation: false
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
model: sonnet
effort: high
context: fork
agent: Explore
---

# Skill 名称

## 任务说明
描述 Claude 应该做什么...

## 步骤
1. 第一步
2. 第二步
3. 第三步

## 约束
- 约束条件...

## 输出格式
- 期望的输出格式...
```
