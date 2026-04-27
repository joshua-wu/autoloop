---
name: autoloop
description: Use when the user wants to set up an autonomous improvement loop with generator/evaluator agents, or resume an existing loop. Triggers on autoloop, 自主循环, autonomous loop, 自动改进, 自动循环.
---

# AutoLoop — 自主循环任务模板

当前版本：1.0.0

## 概述

AutoLoop 是一套三角分工（调度器+生成器+评估器）的自主循环工作模板。此 skill 负责初始化和路由，循环逻辑在生成的 program.md 文件中。

## 路由逻辑

检查当前工作目录，判断走哪条路径：

### 路径判断

1. 检查当前目录是否是 Git 仓库的**根目录**：
   ```bash
   [ "$(git rev-parse --show-toplevel 2>/dev/null)" = "$(pwd)" ]
   ```
   注意：不能仅用 `git rev-parse --is-inside-work-tree`，因为在 Git 仓库的子目录中也会返回 true，但 AutoLoop 需要独立的 Git 仓库根目录来管理 worktree 和 .gitignore。

2. 读取当前目录下的 `program.md`
3. 检查文件头部是否包含 `<!-- autoloop-version: ... -->`

```
当前目录不是 Git 仓库根目录？
  → 询问用户："当前目录不是 Git 仓库根目录，是否自动初始化？（git init）"
  → 用户同意 → 执行 git init + 创建 .gitignore（含 _run/、run.log）
  → 用户拒绝 → 提示"AutoLoop 需要独立的 Git 仓库来管理版本，请在仓库根目录或新目录中运行"，退出

是 Git 仓库根目录 + 有 autoloop-version 标记 → 走"恢复/继续"路径
是 Git 仓库根目录 + 没有或文件不存在       → 走"首次初始化"路径
```

---

## 路径 A：首次初始化

当前目录没有 AutoLoop 模板文件。执行以下步骤：

### Step 1：交互式引导

逐个询问用户（使用 AskUserQuestion 工具，每次一个问题）。前两部分最为关键——用户的描述越详细，生成的 task.md 质量越高，循环效果越好。

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

将 `program.md`、`generator.md`、`evaluator.md` 写入当前工作目录。每个文件头部加入版本标记：

```
<!-- autoloop-version: 1.0.0 -->
```

根据用户在 Step 1 的回答，生成 `config.md` 写入当前工作目录。以 `templates/config.md` 为参考格式，填入用户的实际配置值。

### Step 3：启动

告诉用户文件已就绪，然后直接读取 `program.md` 并按其中的 Setup 流程执行。

---

## 路径 B：恢复/继续

当前目录已有 AutoLoop 模板文件。

### Step 1：版本检查

从 `program.md` 头部提取版本号，与 skill 内置版本（1.0.0）对比：
- **匹配** → 继续
- **不匹配** → 告诉用户："模板文件版本为 vX.X.X，当前 skill 版本为 v1.0.0。是否要更新模板文件？（config.md 和 _run/ 目录会保留）"
  - 用户同意 → 重新写入 program.md、generator.md、evaluator.md（保留 config.md 和 _run/）
  - 用户拒绝 → 使用当前文件继续

### Step 2：状态检查

检查 `_run/dispatcher/state.json` 是否存在：
- **存在** → 告诉用户"检测到上次中断的进度（第 N 轮，phase: XXX），将从中断处恢复"
- **不存在** → 告诉用户"将开始新的循环"

### Step 3：启动

读取 `program.md` 并按其中的逻辑执行（Setup 或 Main Loop）。

---

## 路径 C：更新模板

如果用户明确要求更新模板（如"更新 autoloop 模板"、"升级模板"）：

1. 从 `templates/` 重新写入 `program.md`、`generator.md`、`evaluator.md`（带新版本号）
2. **保留** `config.md` 和 `_run/` 目录（不丢失用户配置和历史）
3. 告诉用户"模板已更新到 v1.0.0，config.md 和历史记录已保留"

---

## 注意事项

- 此 skill 只负责初始化和路由，**不包含循环逻辑**
- 循环逻辑在 `program.md` 中，由主会话读取并执行
- skill 执行完毕后，应让主会话接管（读取 program.md）
