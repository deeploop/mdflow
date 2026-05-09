# MDFlow 技术调研报告

**作者：** AI 技术分析  
**日期：** 2026年5月9日  
**当前版本：** v2.35.5

---

## 一、项目概述

**MDFlow** 是由 [John Lindquist](https://github.com/johnlindquist) 开发的开源 CLI 工具，核心理念是将 Markdown 文件变成可执行的 AI 提示词脚本（"可执行 Markdown"）。用户无需编写代码，直接在 `.md` 文件中编写提示词，通过命令行驱动 Claude、Gemini、Codex 或 GitHub Copilot 执行任务。

- **GitHub：** [github.com/johnlindquist/mdflow](https://github.com/johnlindquist/mdflow)
- **官网：** [mdflow.dev](https://mdflow.dev/)
- **VS Code 插件：** [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=johnlindquist.mdflow)
- **许可证：** 开源（MIT）

---

## 二、核心设计哲学

MDFlow 遵循 **Unix 哲学**：

- **透明**：不隐藏参数，所有 frontmatter 键直接映射为 CLI flags
- **可组合**：通过 stdin/stdout 管道串联多个 AI 代理
- **零配置**：通过文件名推断 AI 后端，无需额外配置文件
- **可复用**：Markdown "配方"（recipe）可存入目录，加入 PATH 全局调用

---

## 三、安装方式

```bash
# 方式一：npm 全局安装
npm install -g mdflow

# 方式二：bun
bun install && bun link
```

安装后同时获得 `mdflow` 和 `md` 两个命令。

---

## 四、文件名约定（后端路由）

文件名格式决定调用哪个 AI 后端：

| 文件名示例 | 调用后端 | 模式 |
|---|---|---|
| `task.claude.md` | Claude Code | 非交互（print）模式 |
| `task.gemini.md` | Gemini CLI | 非交互模式 |
| `task.codex.md` | OpenAI Codex | 非交互模式 |
| `task.copilot.md` | GitHub Copilot | 静默模式 |
| `task.i.claude.md` | Claude Code | **交互模式**（`.i.` 标记） |

也可通过 CLI flag 覆盖：`md task.md --_command gemini`

---

## 五、Frontmatter 配置系统

在 `.md` 文件顶部写 YAML frontmatter，所有非系统键**自动转换为对应 AI CLI 的 flags**。

### 5.1 系统保留键（下划线前缀）

| 键名 | 说明 |
|---|---|
| `_varname` | 定义模板变量及默认值 |
| `_inputs` | 交互式表单字段（文本、选项、数字、确认、密码） |
| `_env` | 注入环境变量 |
| `_interactive` / `_i` | 启用交互模式 |
| `_subcommand` | 前置子命令 |
| `_cwd` | 覆盖工作目录 |
| `context_window` | Token 上限覆盖 |

### 5.2 透传 flags 示例

```yaml
---
model: opus                          # → --model opus
dangerously-skip-permissions: true   # → --dangerously-skip-permissions
mcp-config: ./mcp.json              # → --mcp-config ./mcp.json
add-dir:                             # → --add-dir ./src --add-dir ./tests
  - ./src
  - ./tests
---
```

---

## 六、模板变量系统

变量在 frontmatter 中以 `_` 前缀声明，在正文中用 `{{ }}` 引用，支持 [LiquidJS](https://liquidjs.com/) 语法（条件、循环、过滤器）。

| 变量 | 含义 |
|---|---|
| `{{ _stdin }}` | 管道输入内容 |
| `{{ _1 }}`、`{{ _2 }}` | 位置参数 |
| `{{ _args }}` | 所有位置参数（编号列表） |
| 自定义变量 | `_feature: "Auth"` → `{{ _feature }}` |

CLI 覆盖：`md create.claude.md --_feature "Payments"`

---

## 七、内容导入与内联执行

### 文件导入（自动格式化为 XML 块）

```
@./src/api.ts              # 单文件
@./src/**/*.ts             # Glob 通配
@./src/api.ts:10-50        # 行范围
@./src/types.ts#UserType   # 符号提取
@https://example.com/doc   # 远程 URL（缓存1小时）
```

### Shell 命令内联

```
当前分支：!`git branch --show-current`
最新差异：!`git diff HEAD~1`
```

---

## 八、CLI 命令与 mdflow 专用 Flags

```bash
md <file.md> [flags]      # 执行 markdown 提示词
md create [name]          # 创建新代理文件
md explain <agent.md>     # 预览解析后的完整配置
md setup                  # 配置 shell 别名
md logs                   # 查看日志目录
```

| mdflow 专用 Flag | 说明 |
|---|---|
| `--_command`, `-_c` | 指定 AI 后端 |
| `--_dry-run` | 预览，不实际执行 |
| `--_interactive`, `-_i` | 启用交互模式 |
| `--_edit` | 执行前在编辑器中打开解析结果 |
| `--_no-cache` | 跳过远程 URL 缓存 |
| `--_context` | 显示上下文树后退出 |
| `--_quiet` | 跳过 dashboard 显示 |
| `--raw` | 原始输出，不渲染 Markdown |

---

## 九、管道与代理链（v2.35.0+）

支持标准 Unix 管道，可将多个 AI 代理串联：

```bash
# 将 foo.md 的输出作为 bar.md 的输入
md foo.claude.md | md bar.gemini.md

# 注入外部数据
cat data.json | md analyze.claude.md

# 多级处理链
md extract.claude.md | md summarize.gemini.md | md format.claude.md
```

---

## 十、全局配置

`~/.mdflow/config.yaml` 中设置各后端默认值：

```yaml
commands:
  claude:
    model: sonnet
  copilot:
    silent: true
```

---

## 十一、上下文 Dashboard

执行前自动展示预检信息：
- **上下文树**：列出所有导入文件和内联内容
- **Token 估算**：预估消耗量和成本
- 通过 `--_quiet` 关闭，或用 `--_context` 仅查看不执行

---

## 十二、日志系统

所有执行记录自动保存至 `~/.mdflow/logs/`，可用 `md logs` 查看目录。

---

## 十三、VS Code 插件

- **插件 ID：** `johnlindquist.mdflow`
- **功能：** 为 AI 代理 Markdown 文件提供**自动补全**，frontmatter 键名提示、变量提示、后端关键词补全
- **安装：** VS Code 扩展市场搜索 "MDFlow"

---

## 十四、最新版本更新亮点（v2.34.0 → v2.35.5）

| 版本 | 关键更新 |
|---|---|
| **v2.35.5** | 修复 spinner 渲染（流式输出前正确停止） |
| **v2.35.4** | 修复逗号分隔的可变参数 flag 拆分 |
| **v2.35.3** | 可变参数 flag 改用 `--flag=value` 格式 |
| **v2.35.2** | 导入支持父目录 Glob 模式 |
| **v2.35.0** | **新增代理间管道**（`foo.md \| bar.md`） |
| **v2.34.0** | 执行前编辑、自愈重试、交互表单、运行后操作菜单、Secret 脱敏、语义代理选择、可视化上下文树与成本 Dashboard |

---

## 十五、与 OpenClaw 集成方案

结合 OpenClaw 多渠道 AI 网关架构，推荐以下集成路径：

### 方案 A：mdflow 作为 OpenClaw Skill（推荐）

```
skills/mdflow/
  index.ts     ← SkillEntry + OpenClawSkillMetadata（npm 安装规格）
  skill.ts     ← dispatch → core tool，调用 md <template.claude.md>
```

代理可通过工具调用触发 `md` 命令，mdflow 负责模板渲染和后端路由。

### 方案 B：Markdown 提示词模板库

在 skill workspace 中维护 `.claude.md` / `.gemini.md` 模板目录，代理按需选择模板执行——适合固定、可复用的结构化任务。

### 方案 C：mdflow stdout 接入 OpenClaw 渠道

Channel 插件将 `md` 作为子进程启动，将其输出注入 OpenClaw 消息分发——适合脚本化、批量自动化场景。

---

## 十六、总结

| 维度 | 评价 |
|---|---|
| **上手难度** | 低——会写 Markdown 即可上手 |
| **可扩展性** | 中——无插件系统，靠管道组合 |
| **多后端支持** | 强——Claude / Gemini / Codex / Copilot |
| **与 OpenClaw 适配度** | 高——作为 Skill 嵌入最自然 |
| **活跃度** | 高——2025年底仍持续发版 |

MDFlow 的核心价值在于**让 AI 工作流以 Markdown 文档形式存在**——可读、可版本控制、可共享。与 OpenClaw 结合后，可为多渠道 AI 网关提供结构化、可复用的提示词执行层。

---

## 参考资料

- [GitHub - johnlindquist/mdflow](https://github.com/johnlindquist/mdflow)
- [mdflow.dev 官网](https://mdflow.dev/)
- [MDFlow VS Code 插件 - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=johnlindquist.mdflow)
- [Releases · johnlindquist/mdflow](https://github.com/johnlindquist/mdflow/releases)
- [My AGENTS.md for MDFlow · GitHub Gist](https://gist.github.com/intellectronica/b89c75f488b68275666ecf9873387d87)
- [Programming in Markdown with MDFlow - Eleanor Berger](https://elite-ai-assisted-coding.dev/p/programming-in-markdown-with-mdflow)
