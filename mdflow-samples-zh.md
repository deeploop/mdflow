# MDFlow 示例代码详解（中文）

**作者：** AI 技术分析  
**日期：** 2026年5月9日  
**当前版本：** v2.35.5

---

## 一、基础入门示例（`examples/`）

---

### 1. `hello.claude.md` — 最简示例

```yaml
---
model: sonnet
print: true
---
Say "Hello from mdflow!" and nothing else.
```

**解析：**
- 文件名 `hello.claude.md` → md 自动推断使用 `claude` 命令
- `model: sonnet` → 透传为 `--model sonnet`
- `print: true` → 透传为 `--print`（非交互模式，输出后退出）
- 正文就是发给 AI 的 prompt

**实际执行的命令：**
```bash
claude --model sonnet --print "Say \"Hello from mdflow!\" and nothing else."
```

---

### 2. `hello.copilot.md` — 无 frontmatter 示例

```markdown
Say "Hello from Copilot!" and nothing else.
```

**解析：**
- **完全没有 frontmatter**，仍然可以运行
- 文件名 `hello.copilot.md` → 推断后端为 `copilot`
- md 自动加入 copilot 的打印模式默认 flags

**实际执行的命令：**
```bash
copilot --silent --prompt "Say \"Hello from Copilot!\" and nothing else."
```

> 要点：不需要任何配置，文件名即配置。

---

### 3. `template-args.claude.md` — 模板变量 + 默认值

```yaml
---
_feature_name: Authentication
_target_dir: src/features
model: sonnet
print: true
---
Create a new feature called "{{ _feature_name }}" in {{ _target_dir }}.
Include:
- A main module file
- Unit tests
- README documentation
```

**解析：**
- `_feature_name` 和 `_target_dir` 是**模板变量**（`_` 前缀 = 系统变量，不传给 claude）
- 双花括号 `{{ }}` 是 LiquidJS 模板语法，执行前被替换
- 默认值写在 frontmatter 中，CLI 可覆盖

**运行方式：**
```bash
# 使用默认值
md template-args.claude.md
# → prompt: Create a new feature called "Authentication" in src/features.

# 覆盖变量
md template-args.claude.md --_feature_name "Payments" --_target_dir "src/billing"
# → prompt: Create a new feature called "Payments" in src/billing.
```

---

### 4. `positional-vars.claude.md` — 位置参数

```yaml
---
print: true
---
Translate "{{ _1 }}" to {{ _2 }}.
```

**解析：**
- `{{ _1 }}`、`{{ _2 }}` 自动映射到命令行位置参数
- 无需在 frontmatter 声明

**运行方式：**
```bash
md positional-vars.claude.md "hello world" "French"
# → prompt: Translate "hello world" to French.
```

---

### 5. `args-list.claude.md` — 多位置参数列表

```yaml
---
print: true
---
Process these items:
{{ _args }}
```

**解析：**
- `{{ _args }}` 会把所有位置参数展开为**编号列表**

**运行方式：**
```bash
md args-list.claude.md "apple" "banana" "cherry"
# → Process these items:
# → 1. apple
# → 2. banana
# → 3. cherry
```

---

### 6. `optional-flags.claude.md` — 条件模板（无需声明变量）

```yaml
---
print: true
---
{% if _mode == "detailed" %}
Provide a detailed, comprehensive analysis.
{% else %}
Provide a brief summary.
{% endif %}

Analyze this codebase.
```

**解析：**
- `_mode` **没有在 frontmatter 声明**，直接在正文中使用
- 用 LiquidJS `{% if %}` 条件语法动态生成不同 prompt
- 不传值时 md 会交互式提示输入，或可通过 flag 传入

**运行方式：**
```bash
md optional-flags.claude.md --_mode detailed   # 详细分析
md optional-flags.claude.md                    # 简要摘要
```

---

### 7. `commit.claude.md` — 管道输入（stdin）

```yaml
---
model: sonnet
print: true
---
Generate a concise, conventional commit message for the following diff.
Use the format: type(scope): description

Types: feat, fix, docs, style, refactor, test, chore

Keep it under 72 characters.

{{ _stdin }}
```

**解析：**
- `{{ _stdin }}` 接收管道传入的内容
- 适合把 `git diff` 输出作为上下文传给 AI

**运行方式：**
```bash
git diff --staged | md commit.claude.md
# AI 读取 diff 内容，生成规范的提交消息
```

---

### 8. `command-inline.claude.md` — 内联 Shell 命令

```yaml
---
model: sonnet
print: true
---
Based on the current git status:
!`git status --short`

And recent commits:
!`git log --oneline -5`

What should I work on next?
```

**解析：**
- `` !`命令` `` 语法在 md 预处理时**立即执行** shell 命令，输出内联进 prompt
- `git status --short` 和 `git log --oneline -5` 的真实输出会被插入 prompt 中发送给 AI

**实际发给 AI 的 prompt 大致为：**
```
Based on the current git status:
M  src/index.ts
?? new-file.ts

And recent commits:
a1b2c3 fix: handle edge case
...

What should I work on next?
```

---

### 9. `env-config.claude.md` — 注入环境变量

```yaml
---
env:
  BASE_URL: https://dev.build
  DEBUG: true
model: sonnet
print: true
---
How do you like my url? !`echo $BASE_URL`
```

**解析：**
- `env:` 键值对作为 flags 透传给底层命令
- `` !`echo $BASE_URL` `` 执行时可读取已注入的环境变量

---

### 10. `line-range.claude.md` — 行范围导入

```yaml
---
model: sonnet
print: true
---
Explain what this section of the CLI runner does:

@./src/cli-runner.ts:136-160
```

**解析：**
- `@文件路径:行号-行号` 只导入指定行范围
- 避免把整个大文件塞进 prompt，精确控制上下文
- 导入内容会被格式化为 XML 块插入 prompt

---

### 11. `symbol-extract.claude.md` — 符号提取导入

```yaml
---
model: sonnet
print: true
---
Explain this interface and suggest improvements:

@./src/types.ts#AgentFrontmatter
```

**解析：**
- `@文件路径#符号名` 只提取指定的 TypeScript 符号（interface/type/function/class 等）
- 精确获取类型定义，无需导入整个文件

---

### 12. `review.claude.md` — Glob 批量导入

```yaml
---
model: opus
print: true
---
Review the following code for:
- Bugs and potential issues
- Security vulnerabilities
- Performance problems
- Code style and best practices

@./src/**/*.ts
```

**解析：**
- `@./src/**/*.ts` 使用 Glob 通配符导入目录下所有 `.ts` 文件
- 自动遵守 `.gitignore`，排除 `node_modules` 等
- 默认限制 100,000 tokens；设置 `MDFLOW_FORCE_CONTEXT=1` 可取消限制

---

### 13. `interactive.i.claude.md` — 交互模式

```yaml
---
model: sonnet
---
Let's have a conversation about this codebase.

@./src/index.ts
```

**解析：**
- 文件名含 `.i.`（`interactive.i.claude.md`）→ 启用交互模式
- 不加 `--print`，claude 保持会话，可持续对话
- 同时导入了 `src/index.ts` 作为初始上下文

**实际执行：**
```bash
claude --model sonnet "<prompt含index.ts内容>"
# 启动交互式 Claude 会话
```

---

### 14. `subcommand.codex.md` — 子命令前置

```yaml
---
_subcommand: exec
full-auto: true
---
Analyze this codebase and suggest improvements.
```

**解析：**
- `_subcommand: exec` → 在命令参数前插入子命令 `exec`
- 最终执行：`codex exec --full-auto "Analyze this codebase..."`

---

### 15. `positional-map.copilot.md` — 位置参数映射为 flag

```yaml
---
$1: prompt
model: gpt-4.1
silent: true
---
Explain this code in simple terms.
```

**解析：**
- `$1: prompt` 把**正文（body）**映射为 `--prompt <body>` flag，而非位置参数
- 某些 CLI 工具要求用 `--prompt` 而非位置参数传入内容时使用此功能

---

## 二、多代理管道示例（`examples/multi-agent/`）

这组示例展示了如何用 `|` 串联多个 AI 代理，每个代理专注一项任务，形成**生产流水线**。

---

### 16. `write-test.claude.md` + `implement.claude.md` — TDD 流水线

```yaml
# write-test.claude.md（写测试）
---
model: opus
print: true
---
I need a TypeScript function that: {{ _1 }}

Don't write the implementation.
Write a comprehensive `vitest` test suite...
```

```yaml
# implement.claude.md（写实现）
---
model: sonnet
print: true
---
Here is a test suite. Write the implementation code that makes these tests pass.
Do not modify the tests.

{{ _stdin }}
```

**运行方式：**
```bash
md write-test.claude.md "parses a cron string" | md implement.claude.md
```

**流程：**
1. `write-test`（opus）接收需求描述 → 生成测试代码
2. 测试代码通过管道传给 `implement`（sonnet）→ 生成实现代码

> 角色分工：opus 负责设计（贵但更强），sonnet 负责实现（快且省钱）

---

### 17. `architect.claude.md` + `security-audit.claude.md` — 设计审查流水线

```yaml
# architect.claude.md（架构师）
---
model: opus
print: true
---
Design a TypeScript interface for: {{ _1 }}
Consider scalability and type safety.
```

```yaml
# security-audit.claude.md（安全审计）
---
model: haiku
print: true
---
Analyze the following code design for security vulnerabilities and edge cases.
Be ruthless.

Input:
{{ _stdin }}
```

**运行方式：**
```bash
md architect.claude.md "User Auth System" | md security-audit.claude.md
```

**流程：**
1. opus 设计 TypeScript 接口
2. haiku 对设计进行安全漏洞审查

---

### 18. `audit.claude.md` + `patch.claude.md` — 安全扫描修复流水线

```yaml
# audit.claude.md（漏洞扫描）
---
model: opus
print: true
---
Review this file for security vulnerabilities (XSS, SQLi, sensitive data exposure).
Output a JSON list of issues found with line numbers and descriptions.
If none, output "CLEAN".

File content:
!`cat {{ _1 }}`
```

```yaml
# patch.claude.md（自动修复）
---
model: sonnet
print: true
---
Here is the source file:
!`cat {{ _1 }}`

Here is the security audit:
{{ _stdin }}

If the audit is "CLEAN", output the original file.
Otherwise, rewrite the code to fix the specific vulnerabilities listed.
Output ONLY the code.
```

**运行方式：**
```bash
md audit.claude.md src/api/user.ts | md patch.claude.md src/api/user.ts
```

**流程：**
1. `audit`：用 `` !`cat {{ _1 }}` `` 读取文件内容 → opus 扫描安全漏洞 → 输出 JSON 漏洞列表
2. `patch`：同时读取源文件和管道传来的审计报告 → sonnet 自动修复漏洞

---

### 19. `changelog.claude.md` + `announcement.claude.md` — 发版公告流水线

```yaml
# changelog.claude.md（提取变更历史）
---
model: sonnet
print: true
---
Analyze these commits and group them into "Features", "Fixes", and "Chore".
Ignore merge commits. Output clean Markdown lists.

!`git log --pretty=format:"%s" {{ _1 }}..HEAD`
```

```yaml
# announcement.claude.md（生成公告）
---
model: sonnet
print: true
---
You are a DevRel expert. Take this technical changelog and write a
punchy, exciting LinkedIn post or Tweet thread announcing the release.
Focus on user value, not just commit messages.

Input:
{{ _stdin }}
```

**运行方式：**
```bash
md changelog.claude.md v1.0.0 | md announcement.claude.md
```

**流程：**
1. `changelog`：用 `git log` 取出 `v1.0.0` 到 HEAD 的所有提交 → AI 整理成分类 Markdown
2. `announcement`：AI 把技术变更日志改写为面向用户的 LinkedIn/推特公告

---

### 20. `audit-legacy.claude.md` — 动态 Glob（变量路径）

```yaml
---
model: sonnet
print: true
---
I am migrating our database. Check these files for deprecated raw SQL queries.

@./{{ _1 }}

If you find `db.query('SELECT...`, suggest the equivalent Prisma ORM syntax.
If the file is already using Prisma, output "CLEAN".
```

**解析：**
- `@./{{ _1 }}` 把**模板变量嵌入导入路径**，路径由 CLI 参数动态决定
- 非常灵活：可以把任意 Glob 模式作为参数传入

**运行方式：**
```bash
md audit-legacy.claude.md "src/db/**/*.ts"
# 扫描 src/db/ 下所有 .ts 文件，查找废弃的 SQL 查询
```

---

### 21. `pr-review.claude.md` — 上下文聚合（diff + 需求 + 规范）

```yaml
---
model: sonnet
print: true
---
You are a senior engineer reviewing a Pull Request.

### The Code Changes
!`gh pr diff {{ _1 }}`

### The Original Requirement
!`gh pr view {{ _1 }} --json body -q .body`

### Our Coding Standards
@./CONTRIBUTING.md

Based on the requirements and our standards, provide a bulleted review.
Flag any security risks immediately.
```

**解析：**
- 同时从**三个来源**聚合上下文：
  1. `` !`gh pr diff` `` → PR 的代码变更（shell 命令内联）
  2. `` !`gh pr view` `` → PR 的需求描述（shell 命令内联）
  3. `@./CONTRIBUTING.md` → 项目编码规范（文件导入）
- 让 AI 综合三者进行代码审查，而非只看 diff

**运行方式：**
```bash
md pr-review.claude.md 123   # 审查第 123 号 PR
```

---

### 22. `scaffold.claude.md` — 参照已有模式生成代码

```yaml
---
model: sonnet
print: true
---
Read this existing component to understand our pattern:
@./src/components/Button.tsx

Now, generate a new component named **{{ _1 }}**.
It should have:
1. The component file
2. A matching test file
3. A storybook file

Output strictly valid code blocks.
```

**解析：**
- 先导入 `Button.tsx` 让 AI 学习项目的代码风格
- 再要求按照同样风格生成新组件（含测试和 Storybook）
- "学样板、生新料"的典型用法

**运行方式：**
```bash
md scaffold.claude.md "DropdownMenu"
# 生成 DropdownMenu 组件、测试文件和 Storybook
```

---

## 三、设计模式总结

通过这些示例，可以归纳出 mdflow 的几种核心使用模式：

| 模式 | 代表示例 | 要点 |
|---|---|---|
| **基础调用** | `hello.claude.md` | 文件名 = 后端，frontmatter = flags |
| **模板变量** | `template-args.claude.md` | `_var` + `{{ var }}`，CLI 覆盖 |
| **管道输入** | `commit.claude.md` | `{{ _stdin }}` 接收 `\|` 传来的数据 |
| **Shell 内联** | `command-inline.claude.md` | `` !`cmd` `` 实时注入命令输出 |
| **精准导入** | `line-range.claude.md` | `:行范围` 或 `#符号名` 精确取代码 |
| **批量导入** | `review.claude.md` | Glob `**/*.ts` 导入整个目录 |
| **多代理流水线** | `write-test \| implement` | `\|` 串联，每个代理专注一件事 |
| **上下文聚合** | `pr-review.claude.md` | 同时内联多个来源的上下文 |
| **动态路径** | `audit-legacy.claude.md` | `@./{{ _1 }}` 路径也可用模板变量 |
