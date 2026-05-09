# MDFlow 技术调研报告（详细版）

**作者：** AI 技术分析  
**日期：** 2026年5月9日  
**当前版本：** v2.35.5

---

## 一、项目概述

**MDFlow**（命令别名 `md`）是由 [John Lindquist](https://github.com/johnlindquist) 开发的开源 CLI 工具，核心理念是将 Markdown 文件变成可执行的 AI 提示词脚本——即"可执行 Markdown"。用户无需编写代码，直接在 `.md` 文件中编写提示词，通过命令行驱动 Claude、Gemini、Codex 或 GitHub Copilot 执行任务。

- **GitHub：** [github.com/johnlindquist/mdflow](https://github.com/johnlindquist/mdflow)
- **官网：** [mdflow.dev](https://mdflow.dev/)
- **VS Code 插件：** [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=johnlindquist.mdflow)
- **许可证：** MIT 开源
- **运行时：** Bun（TypeScript 原生，无需编译）

---

## 二、核心设计哲学

MDFlow 遵循 **Unix 哲学**：

- **透明（Transparent）**：不隐藏参数，所有 frontmatter 键直接映射为 CLI flags，执行命令可追溯
- **可组合（Composable）**：通过 stdin/stdout 管道串联多个 AI 代理
- **零配置（Zero-config）**：通过文件名推断 AI 后端，无需额外配置文件
- **可复用（Reusable）**：Markdown "配方"（recipe）可存入目录，加入 PATH 全局调用
- **无魔法映射（No magic）**：frontmatter 键 1:1 传递给底层命令，行为可预测

---

## 三、安装方式

```bash
# 方式一：npm 全局安装（推荐）
npm install -g mdflow

# 方式二：从源码安装
bun install && bun link
```

安装后同时获得 `mdflow` 和 `md` 两个命令，两者完全等价。

### Shell 集成（可选）

运行一次性配置，使 `.md` 文件可直接作为命令执行：

```bash
mdflow setup
```

完成后可直接输入文件名运行：

```bash
review.claude.md --verbose   # 不需要 md 前缀
```

手动配置（zsh）：

```bash
# 加入 ~/.zshrc
alias -s md='mdflow'
export PATH="$HOME/agents:$PATH"   # 你的代理库目录
```

---

## 四、文件名约定（后端路由）

文件名格式 `任务名.后端名.md` 决定调用哪个 AI 后端：

| 文件名示例 | 调用后端 | 实际命令 |
|---|---|---|
| `task.claude.md` | Claude Code | `claude --print "..."` |
| `task.gemini.md` | Gemini CLI | `gemini "..."` |
| `task.codex.md` | OpenAI Codex | `codex exec "..."` |
| `task.copilot.md` | GitHub Copilot | `copilot --silent --prompt "..."` |
| `task.opencode.md` | OpenCode | `opencode run "..."` |
| `task.droid.md` | Droid | `droid exec "..."` |
| `task.i.claude.md` | Claude Code | `claude "..."` **（交互模式）** |

`.i.` 标记插在后端名前，启用交互模式（不加 `--print`）。

也可通过 CLI flag 在运行时覆盖后端：

```bash
md task.md --_command gemini    # 用 gemini 替代文件名推断的后端
md task.md -_c claude           # 短形式
```

---

## 五、Frontmatter 配置系统

在 `.md` 文件顶部写 YAML frontmatter，所有**非系统键**自动转换为对应 AI CLI 的 flags。

### 5.1 系统保留键（下划线前缀，由 md 消费）

| 键名 | 类型 | 说明 |
|---|---|---|
| `_varname` | string | 定义模板变量及默认值（`{{ _varname }}` 引用） |
| `_inputs` | object/array | 交互式表单字段（见第六节） |
| `_env` | object | 在执行前注入环境变量到 `process.env` |
| `_interactive` / `_i` | boolean | 启用交互模式 |
| `_subcommand` | string/string[] | 在 CLI 参数前插入子命令 |
| `_cwd` | string | 覆盖内联命令（`` !`cmd` ``）的工作目录 |
| `context_window` | number | 覆盖 Token 上限（默认按模型自动判断） |
| `$1`, `$2`... | string | 将位置参数映射到指定 flag |

### 5.2 透传 flags 示例

```yaml
---
model: opus                          # → --model opus
dangerously-skip-permissions: true   # → --dangerously-skip-permissions
mcp-config: ./mcp.json              # → --mcp-config ./mcp.json
add-dir:                             # → --add-dir ./src --add-dir ./tests
  - ./src
  - ./tests
p: true                              # → -p（单字符 = 短 flag）
verbose: false                       # 值为 false → 省略（不传）
---
```

**值转换规则：**

| frontmatter 值 | CLI 输出 |
|---|---|
| `key: "value"` | `--key value` |
| `key: true` | `--key` |
| `key: false` | （省略） |
| `key: [a, b]` | `--key=a --key=b` |

### 5.3 位置参数映射（`$N`）

将位置参数或正文体映射到指定 flag：

```yaml
---
$1: prompt    # 文件正文作为 --prompt <body> 而非位置参数
---
```

---

## 六、变量与表单系统

### 6.1 模板变量（`_varname`）

变量在 frontmatter 中以 `_` 前缀声明，正文中用 `{{ }}` 引用，底层使用 [LiquidJS](https://liquidjs.com/) 模板引擎（支持条件、循环、过滤器）。

| 变量 | 含义 |
|---|---|
| `{{ _stdin }}` | 管道输入内容 |
| `{{ _1 }}`、`{{ _2 }}` | 命令行位置参数 |
| `{{ _args }}` | 所有位置参数（编号列表：1. arg1 ...） |
| 自定义变量 | `_feature: "Auth"` → `{{ _feature }}` |

CLI 覆盖：`md create.claude.md --_feature "Payments"`

**无需声明即可使用：** 若变量在正文中出现但未提供，md 会交互式提示输入。

**变量历史（Variable History）：** 上次输入的值会保存至 `~/.mdflow/variable-history.json`，下次运行时作为默认值显示，回车直接接受。用 `--_no-history` 跳过此功能。

### 6.2 交互式表单（`_inputs`）

`_inputs` 支持结构化表单定义，执行前逐字段提示用户输入：

```yaml
---
model: sonnet
_inputs:
  _name:
    type: text
    description: "输入你的名字"
    default: "World"
  _env:
    type: select
    options: [dev, staging, prod]
  _count:
    type: number
    description: "数量？"
  _confirm:
    type: confirm
    description: "确认执行？"
  _secret:
    type: password
    description: "API Key"
---
你好 {{ _name }}！部署到 {{ _env }}，共 {{ _count }} 个。
```

**支持的字段类型：**

| 类型 | 行为 |
|---|---|
| `text` | 自由文本输入（默认） |
| `select` | 从 `options` 列表中选择 |
| `number` | 数字输入 |
| `confirm` | 是/否布尔确认 |
| `password` | 隐藏输入（API Key 等） |

兼容旧格式：`_inputs: [_name, _value]`（仅变量名数组）仍可使用。

### 6.3 LiquidJS 高级用法

```liquid
{% if _force %}--force{% endif %}
{{ _name | upcase }}
{{ _value | default: "fallback" }}
{% for item in _items %}- {{ item }}{% endfor %}
```

---

## 七、内容导入与内联执行

### 7.1 文件导入（自动格式化为 XML 块）

```
@./src/api.ts              # 单文件
@./src/**/*.ts             # Glob 通配（自动遵守 .gitignore）
@./src/api.ts:10-50        # 行范围（仅取第10-50行）
@./src/types.ts#UserType   # 符号提取（interface/type/function/class/const/enum）
@~/path/to/file.md         # ~ 展开为 HOME 目录
@https://example.com/doc   # 远程 URL（缓存1小时，支持 markdown 和 JSON）
```

导入文件以 XML 格式内嵌：

```xml
<api path="src/api.ts">
...文件内容...
</api>
```

**Glob 限制：** 默认最多 100,000 tokens；设置 `MDFLOW_FORCE_CONTEXT=1` 可跳过限制。

**递归导入：** 被导入的文件内部可以继续使用 `@` 导入其他文件。

**代码块保护：** 处于 ` ``` ` 或 `` ` `` 内的 `@` 路径不会被解析。

### 7.2 Shell 命令内联

```markdown
当前分支：!`git branch --show-current`
最近提交：!`git log --oneline -5`
```

反引号内的 shell 命令在 md 预处理时执行，输出被内联到 prompt 中，再发送给 AI 后端。

### 7.3 远程 URL 执行与信任机制（TOFU）

可直接执行远程 Markdown 代理：

```bash
md https://example.com/agent.claude.md
```

首次访问远程 URL 时，md 会显示信任提示（Trust On First Use）。使用 `--_trust` 绕过提示；`--_no-cache` 强制刷新缓存（默认1小时 TTL，缓存路径 `~/.mdflow/cache/`）。

---

## 八、CLI 命令与 mdflow 专用 Flags

### 8.1 子命令

```bash
md <file.md> [flags]      # 执行 markdown 提示词
md create [name]          # 创建新代理文件（自动在 $EDITOR 打开）
md explain <agent.md>     # 预览解析后的完整配置（不执行）
md setup                  # 配置 shell 别名和 PATH
md logs                   # 查看日志目录路径
md help                   # 显示帮助
```

无参数直接运行 `md` 会启动**交互式代理选择器**（按使用频率+新鲜度排序，即 frecency）。

### 8.2 mdflow 专用 Flags（消费后不传给底层命令）

| Flag | 短形式 | 说明 |
|---|---|---|
| `--_command` | `-_c` | 指定 AI 后端（覆盖文件名推断） |
| `--_dry-run` | | 预览将执行的命令，不实际执行 |
| `--_interactive` | `-_i` | 启用交互模式 |
| `--_edit` | | 执行前在 `$EDITOR` 中打开解析结果 |
| `--_no-cache` | | 跳过远程 URL 缓存，强制重新拉取 |
| `--_trust` | | 绕过 TOFU 远程 URL 信任提示 |
| `--_context` | | 显示上下文树后退出（不执行） |
| `--_quiet` | | 跳过执行前 dashboard 显示 |
| `--_no-history` | | 跳过变量历史的读取和保存 |
| `--raw` | | 原始输出，不渲染 Markdown（便于管道传输） |

### 8.3 `md explain` 命令详解

```bash
md explain review.claude.md
```

输出以下信息（不执行）：
- 解析后的命令及来源（文件名/环境变量/flag）
- 合并后的完整 flags（内置默认 → 全局配置 → 项目配置 → frontmatter）
- 展开后的 prompt 预览（含 token 数量估算）
- 远程 URL 的信任状态
- 将注入的环境变量列表

---

## 九、打印模式与交互模式

所有命令默认为**打印（非交互）模式**，执行完毕后退出。`.i.` 文件名标记、`_interactive` frontmatter 或 CLI flag 均可切换为交互模式。

### 各后端打印模式对比

| 后端 | 打印模式命令 | 交互模式命令 |
|---|---|---|
| `claude` | `claude --print "..."` | `claude "..."` |
| `copilot` | `copilot --silent --prompt "..."` | `copilot --silent --interactive "..."` |
| `codex` | `codex exec "..."` | `codex "..."` |
| `gemini` | `gemini "..."` | `gemini --prompt-interactive "..."` |
| `opencode` | `opencode run "..."` | `opencode "..."` |
| `droid` | `droid exec "..."` | `droid "..."` |

---

## 十、管道与代理链（v2.35.0+）

支持标准 Unix 管道，可将多个 AI 代理串联：

```bash
# foo.md 的输出作为 bar.md 的 {{ _stdin }} 输入
md foo.claude.md | md bar.gemini.md

# 注入外部数据
cat data.json | md analyze.claude.md

# 多级处理链
md extract.claude.md | md summarize.gemini.md | md format.claude.md

# 只取前5行（EPIPE 被优雅处理）
md task.claude.md | head -n 5
```

管道时使用 `--raw` 跳过 Markdown 渲染，输出纯文本：

```bash
md task.claude.md --raw | jq .
```

---

## 十一、全局配置

`~/.mdflow/config.yaml` 中为各后端设置默认 frontmatter：

```yaml
commands:
  claude:
    model: sonnet       # claude 默认模型
  copilot:
    silent: true        # copilot 始终 --silent
```

**配置优先级（低→高）：** 内置默认 → `~/.mdflow/config.yaml` → 项目级 frontmatter → CLI flags

---

## 十二、环境变量加载

md 自动从 Markdown 文件所在目录加载 `.env` 文件，加载顺序如下（后加载的覆盖先加载的）：

1. `.env`
2. `.env.local`
3. `.env.development` / `.env.production`（按 `NODE_ENV` 决定）
4. `.env.development.local` / `.env.production.local`

环境变量在内联命令（`` !`echo $API_KEY` ``）和底层 AI 命令的子进程中均可使用。

全局环境变量：

| 变量 | 说明 |
|---|---|
| `MDFLOW_FORCE_CONTEXT` | 设为 `1` 禁用 Glob 导入的 100k token 上限 |
| `NODE_ENV` | 控制加载哪个 `.env.[NODE_ENV]` 文件（默认 `development`） |
| `MA_COMMAND` | 全局覆盖 AI 后端命令 |

---

## 十三、上下文 Dashboard 与 Token 估算

执行前自动展示预检信息（使用 `--_quiet` 关闭，`--_context` 仅查看不执行）：

```
┌─ Pre-Flight ──────────────────────────────────────────────────┐
│  📄 review.claude.md                                   1.2 KB │
│  ├── 📁 @./src/**/*.ts                     (12 files) 24.5 KB │
│  └── 📄 @./README.md                                   3.1 KB │
│                                                               │
│  Total: 28.8 KB (~7,200 tokens)                              │
└───────────────────────────────────────────────────────────────┘
```

Dashboard 展示：
- **上下文树**：列出所有导入文件、Glob 展开结果、内联命令输出
- **Token 估算**：预估消耗量（基于 tiktoken 分词）
- **成本预估**：基于所选模型单价计算大致费用

---

## 十四、执行前编辑（Edit-Before-Execute）

```bash
md task.claude.md --_edit
```

在 `$EDITOR` 中打开**完全解析后**的 prompt（模板替换和导入展开均已完成），可手动调整后保存再执行，适合需要临时修改 prompt 的场景。

---

## 十五、自愈重试（Auto-Heal Retry）

底层命令执行失败时，md 自动触发重试循环，尝试恢复（v2.34.0 新增）。

---

## 十六、运行后操作菜单（Post-Run Menu）

命令执行完毕后，md 展示操作菜单（v2.34.0 新增），可选择：
- 复制输出到剪贴板
- 将输出写入文件
- 重新执行
- 退出

---

## 十七、Secret 脱敏（Secret Masking）

md 在日志和 dashboard 中自动检测并隐藏敏感信息（API Key、Token 等），防止凭证泄露（v2.34.0 新增）。

---

## 十八、语义代理选择器（Semantic Agent Picker）

无参数运行 `md` 时，启动交互式选择器：
- 按 **frecency**（频率 × 新鲜度）排序，常用代理优先显示
- 支持内容搜索（Tab/Shift+Tab 在匹配项间跳转）
- 显示代理描述（从 frontmatter 或文件首行提取）

---

## 十九、Ad-hoc 一次性执行

通过 `md.command` 别名直接执行一次性 prompt，无需创建文件：

```bash
md.claude "解释这段代码"
md.gemini "翻译成中文"
```

---

## 二十、富文本终端渲染

LLM 输出默认通过 `marked-terminal` 渲染为带语法高亮的 Markdown，包含：
- 标题层级样式
- 代码块语法高亮
- 列表、表格等格式

使用 `--raw` 跳过渲染（用于管道传输）：

```bash
md task.claude.md --raw | jq .
```

---

## 二十一、日志系统

所有执行记录自动保存至 `~/.mdflow/logs/<agent-name>/`（使用 pino 结构化日志）：

```bash
md logs    # 查看日志目录路径
```

---

## 二十二、VS Code 插件

- **插件 ID：** `johnlindquist.mdflow`
- **功能：** 为 `.claude.md`、`.gemini.md` 等文件提供智能补全：
  - frontmatter 键名提示
  - 模板变量提示（`{{ _` 触发）
  - AI 后端关键词补全
- **安装：** VS Code 扩展市场搜索 "MDFlow"

---

## 二十三、技术架构详解

### 核心执行流程（`src/index.ts` → `CliRunner`）

```
.md 文件 / 远程 URL
    │
    ▼
parseFrontmatter()          ← 解析 YAML frontmatter + body
    │
    ▼
resolveCommand()            ← MA_COMMAND > --_command flag > 文件名推断
    │
    ▼
loadGlobalConfig()          ← ~/.mdflow/config.yaml 默认值
    │
    ▼
applyDefaults()             ← 合并内置默认 + 全局配置
    │
    ▼
applyInteractiveMode()      ← 根据文件名 .i. 或 flag 切换模式
    │
    ▼
expandImports()             ← 解析 @path、!`cmd`、URL，展开内容
    │
    ▼
substituteTemplateVars()    ← LiquidJS 渲染（变量、条件、循环）
    │
    ▼
buildArgs()                 ← frontmatter → CLI flags
    │
    ▼
runCommand()                ← spawn 子进程，流式输出，渲染 Markdown
```

### 关键模块

| 模块 | 职责 |
|---|---|
| `command.ts` | 命令解析、flag 构建、子进程执行 |
| `config.ts` | 全局配置加载、默认值合并 |
| `imports.ts` | 文件/Glob/行范围/符号/URL/命令 导入解析 |
| `template.ts` | LiquidJS 模板渲染 |
| `history.ts` | 变量历史持久化（frecency + 变量值） |
| `secrets.ts` | 敏感信息检测与脱敏 |
| `ui.ts` | 交互式选择器、表单输入 |
| `streams.ts` | 流式输出处理 |
| `spinner.ts` | 执行中 spinner 动画 |
| `logger.ts` | pino 结构化日志 |
| `limits.ts` | Token 上限控制 |
| `trust.ts` | 远程 URL 信任管理（TOFU） |
| `explain.ts` | `md explain` 功能实现 |
| `create.ts` | `md create` 模板生成 |
| `edit-prompt.ts` | `--_edit` 执行前编辑 |
| `process-manager.ts` | 进程生命周期管理（信号、光标恢复） |
| `schema.ts` | Zod 最小验证（系统键 + passthrough） |

---

## 二十四、最新版本更新亮点

| 版本 | 关键更新 |
|---|---|
| **v2.35.5** | 修复 spinner 渲染（流式输出前正确停止） |
| **v2.35.4** | 修复逗号分隔的可变参数 flag 拆分 |
| **v2.35.3** | 可变参数 flag 改用 `--flag=value` 格式 |
| **v2.35.2** | 导入支持父目录 Glob 模式（#13） |
| **v2.35.0** | **新增代理间管道**（`foo.md \| bar.md`） |
| **v2.34.0** | 执行前编辑（`--_edit`）、自愈重试、交互式表单（`_inputs`）、运行后操作菜单、Secret 脱敏、语义代理选择器、可视化上下文树与成本 Dashboard、变量历史持久化 |
| **v2.33.0** | spinner 显示命令预览而非文件名 |
| **v2.32.0** | `md create` 后自动在编辑器打开 |

---

## 二十五、与 OpenClaw 集成方案

结合 OpenClaw 多渠道 AI 网关架构，推荐以下集成路径：

### 方案 A：mdflow 作为 OpenClaw Skill（推荐）

```
skills/mdflow/
  index.ts     ← SkillEntry + OpenClawSkillMetadata
  skill.ts     ← dispatch → core tool → 调用 md <template.claude.md>
```

代理通过工具调用触发 `md` 命令，mdflow 负责模板渲染和后端路由，OpenClaw 获得结构化输出。

### 方案 B：Markdown 提示词模板库

在 skill workspace 中维护 `.claude.md` / `.gemini.md` 模板目录，代理按需选择模板执行——适合固定、可复用的结构化任务。

### 方案 C：mdflow stdout 接入 OpenClaw 渠道

Channel 插件将 `md` 作为子进程启动，将其输出注入 OpenClaw 消息分发——适合脚本化、批量自动化场景。

---

## 二十六、构建代理库（Agent Library）

推荐在 `~/agents/` 目录统一管理代理文件，加入 PATH：

```
~/agents/
├── review.claude.md     # 代码审查
├── commit.gemini.md     # 生成提交消息
├── explain.claude.md    # 代码解释
├── test.codex.md        # 测试生成
└── debug.claude.md      # 调试助手
```

```bash
export PATH="$HOME/agents:$PATH"

# 从任意目录使用
review.claude.md
git diff | commit.gemini.md
```

---

## 二十七、总结

| 维度 | 评价 |
|---|---|
| **上手难度** | 低——会写 Markdown 即可上手 |
| **可扩展性** | 中——无插件系统，靠管道和模板组合 |
| **多后端支持** | 强——Claude / Gemini / Codex / Copilot / OpenCode / Droid |
| **工程化程度** | 高——变量历史、Secret 脱敏、Token 估算、日志、TOFU |
| **与 OpenClaw 适配度** | 高——作为 Skill 嵌入最自然 |
| **活跃度** | 高——2025年底仍持续高频发版 |

MDFlow 的核心价值在于**让 AI 工作流以 Markdown 文档形式存在**——可读、可版本控制、可共享、可管道组合。v2.34.0 起新增的交互式表单、Secret 脱敏、上下文 Dashboard 等功能使其从简单的"prompt 包装器"升级为完整的 AI 工作流运行时。

---

## 参考资料

- [GitHub - johnlindquist/mdflow](https://github.com/johnlindquist/mdflow)
- [mdflow.dev 官网](https://mdflow.dev/)
- [MDFlow VS Code 插件 - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=johnlindquist.mdflow)
- [Releases · johnlindquist/mdflow](https://github.com/johnlindquist/mdflow/releases)
- [My AGENTS.md for MDFlow · GitHub Gist](https://gist.github.com/intellectronica/b89c75f488b68275666ecf9873387d87)
- [Programming in Markdown with MDFlow - Eleanor Berger](https://elite-ai-assisted-coding.dev/p/programming-in-markdown-with-mdflow)
