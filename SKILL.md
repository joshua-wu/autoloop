---
name: autoloop
description: Use when the user wants to set up an autonomous improvement loop with generator/evaluator agents, or resume an existing loop. Triggers on autoloop, 自主循环, autonomous loop, 自动改进, 自动循环.
---

# AutoLoop — 自主循环任务模板

当前版本：1.1.0

## 概述

AutoLoop 是一套三角分工（调度器+生成器+评估器）的自主循环工作模板。此 skill 负责初始化和路由，循环逻辑在生成的 program.md 文件中。

同一个目录下可以运行多个独立的 autoloop 任务，每个任务有自己的配置文件和运行时目录。

## 任务命名约定

| 配置文件 | 任务名 | 运行目录 | 分支前缀 |
|---------|--------|---------|---------|
| `config.md` | (默认，无名) | `_run/` | `autoloop/<tag>` |
| `config-<task>.md` | `<task>` | `_run/<task>/` | `autoloop/<task>/<tag>` |

## 路由逻辑

检查当前工作目录，判断走哪条路径：

### Step 0：环境检查

1. 检查当前目录是否是 Git 仓库的**根目录**：
   ```bash
   [ "$(git rev-parse --show-toplevel 2>/dev/null)" = "$(pwd)" ]
   ```
   注意：不能仅用 `git rev-parse --is-inside-work-tree`，因为在 Git 仓库的子目录中也会返回 true，但 AutoLoop 需要独立的 Git 仓库根目录来管理 worktree 和 .gitignore。

   ```
   当前目录不是 Git 仓库根目录？
     → 询问用户："当前目录不是 Git 仓库根目录，是否自动初始化？（git init）"
     → 用户同意 → 执行 git init + 创建 .gitignore（含 _run/、run.log）
     → 用户拒绝 → 提示"AutoLoop 需要独立的 Git 仓库来管理版本，请在仓库根目录或新目录中运行"，退出
   ```

### Step 1：任务发现

扫描当前目录下的配置文件：

```bash
ls config.md config-*.md 2>/dev/null
```

根据结果分流：

- **0 个配置文件** → 走路径 A（首次初始化）
- **1 个配置文件** → 自动识别任务名，走路径 B（恢复/继续）或路径 A（如果没有 program.md）
- **2+ 个配置文件** → 列出所有任务，询问用户运行哪一个（或从 `/autoloop` 参数中读取任务名）

**任务名推导**：
- 文件名为 `config.md` → 任务名为空（默认任务）
- 文件名为 `config-<task>.md` → 任务名为 `<task>`

### Step 2：路径判断

确定任务后：
- 检查 `program.md` 是否存在且包含 `<!-- autoloop-version: ... -->`
- 有版本标记 → 走路径 B（恢复/继续）
- 无版本标记或文件不存在 → 走路径 A（首次初始化）

---

## 路径 A：首次初始化

### Step 1：交互式引导

逐个询问用户（使用 AskUserQuestion 工具，每次一个问题）。前两部分最为关键——用户的描述越详细，生成的 task.md 质量越高，循环效果越好。

**第零部分：任务命名（仅当目录下已有其他 config 文件时）**

0. **任务名**：给这个任务取个名字？（如：doc-optimize、bugfix-auth）
   - 名字会用于配置文件名（`config-<name>.md`）和运行目录（`_run/<name>/`）
   - 只允许字母、数字、短横线
   - 如果目录下没有其他 config 文件，可以跳过（使用默认的 `config.md`）

**第一部分：任务描述（需要产出什么）**

1. **角色**：你希望 AI 扮演什么角色？（如：文档优化师、ML 研究员、Bug 修复工程师）
2. **任务目标**：你希望最终产出什么样的结果？请尽可能具体描述：
   - 不好的例子：「改善代码」
   - 好的例子：「让 csv_analyzer.py 通过所有 pytest 测试，修复资源泄漏、排序逻辑、异常处理等问题」
   - 引导用户思考：最终"做好了"是什么样子？交付标准是什么？
3. **目标文件**：哪些文件可以被修改？（列出文件或目录路径）
4. **只读文件**：哪些文件作为参考但不能修改？（可选）

**第二部分：质量要求（怎么判断做得好不好）**

5. **评估方法**：用什么方式判断改动的好坏？可以组合多种方式：
   - 跑命令（如 `pytest`、`npm test`、`vale`）——给出具体命令
   - 读文件分析内容质量——关注哪些方面？（准确性？可读性？完整性？）
   - 对比源码/参考文件——对比什么？（数据准确性？API 一致性？）
   - 引导用户思考：如果你自己来检查，你会看什么指标、检查什么内容？
6. **质量底线**：有没有绝对不能出现的问题？（如：测试不能回归、不能引入安全漏洞、不能删除已有功能）。这些会成为评估器的 Critical 判定标准。

**第三部分：策略配置**

7. **生成策略**：每轮改动的粒度？
   - `one_change`（默认）— 每轮一个聚焦改动（推荐大多数场景）
   - `multi_change` — 每轮可做多个相关改动（适合 bug 修复、重构）
   - `exploratory` — 大胆的探索性改动（适合架构实验）
8. **评估深度**：
   - `quantitative` — 以量化指标为主（有明确的数值指标时选择）
   - `qualitative` — 以定性分析为主（内容质量、可读性类任务选择）
   - `mixed`（默认）— 两者结合

### Step 2：生成模板文件

读取 skill 目录下 `templates/` 中的 4 个模板文件：
- `templates/program.md`
- `templates/generator.md`
- `templates/evaluator.md`
- `templates/config.md`

**如果 program.md、generator.md、evaluator.md 不存在**，写入当前工作目录，每个文件头部加入版本标记：
```
<!-- autoloop-version: 1.1.0 -->
```

**如果已存在**（多任务场景下前一个任务已创建），跳过——这些模板是所有任务共享的。

根据用户在 Step 1 的回答，生成配置文件：
- 无任务名 → 写入 `config.md`
- 有任务名 → 写入 `config-<task>.md`

以 `templates/config.md` 为参考格式，填入用户的实际配置值。

### Step 3：启动

告诉用户文件已就绪，然后直接读取 `program.md` 并按其中的 Setup 流程执行。传入配置文件路径（如 `config-doc-optimize.md`）。

---

## 路径 B：恢复/继续

当前目录已有 AutoLoop 模板文件和对应的配置文件。

### Step 1：版本检查

从 `program.md` 头部提取版本号，与 skill 内置版本（1.1.0）对比：
- **匹配** → 继续
- **不匹配** → 告诉用户："模板文件版本为 vX.X.X，当前 skill 版本为 v1.1.0。是否要更新模板文件？（配置文件和 _run/ 目录会保留）"
  - 用户同意 → 重新写入 program.md、generator.md、evaluator.md（保留所有 config*.md 和 _run/）
  - 用户拒绝 → 使用当前文件继续

### Step 2：状态检查

确定运行目录（从任务名推导）后，检查对应的 state.json：
- 默认任务 → 检查 `_run/dispatcher/state.json`
- 命名任务 → 检查 `_run/<task>/dispatcher/state.json`

结果：
- **存在** → 告诉用户"检测到上次中断的进度（第 N 轮，phase: XXX），将从中断处恢复"
- **不存在** → 告诉用户"将开始新的循环"

### Step 3：启动

读取 `program.md` 并按其中的逻辑执行（Setup 或 Main Loop）。传入配置文件路径。

---

## 路径 C：更新模板

如果用户明确要求更新模板（如"更新 autoloop 模板"、"升级模板"）：

1. 从 `templates/` 重新写入 `program.md`、`generator.md`、`evaluator.md`（带新版本号）
2. **保留** 所有 `config*.md` 文件和 `_run/` 目录（不丢失用户配置和历史）
3. 告诉用户"模板已更新到 v1.1.0，配置文件和历史记录已保留"

---

## 注意事项

- 此 skill 只负责初始化和路由，**不包含循环逻辑**
- 循环逻辑在 `program.md` 中，由主会话读取并执行
- skill 执行完毕后，应让主会话接管（读取 program.md）
- `program.md`、`generator.md`、`evaluator.md` 是所有任务共享的模板，不按任务区分
- 每个任务的差异体现在各自的 `config[-<task>].md` 和 `_run/[<task>/]` 目录中
