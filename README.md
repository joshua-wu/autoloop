# autoloop

**把任何任务变成自主改进循环。睡前启动，醒来收获。**

[English](README_EN.md) | 中文

autoloop 是一套通用的三角分工自主循环框架，灵感来自 Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch)。它将"修改-评估-决策"的循环抽象成可复用的模板，让 AI agent 能够自主、持续、安全地改进任何目标文件——无论是训练代码、API 文档还是配置文件。

## 工作原理

```
                    ┌─────────────┐
                    │  Dispatcher  │
                    │   (调度器)    │
                    │  决策 + 记录  │
                    └──┬───────┬──┘
               keep/   │       │   评估报告
              discard  │       │
                ┌──────┴──┐ ┌──┴──────┐
                │Generator│ │Evaluator│
                │ (生成器) │ │ (评估器) │
                │ 修改文件 │ │ 找出问题 │
                └─────────┘ └─────────┘
                      ↑           ↑
                      │  worktree  │
                      │   隔离     │
                      └─────┬─────┘
                            │
                     ┌──────┴──────┐
                     │  目标文件    │
                     │  (你的产出)  │
                     └─────────────┘
```

每一轮循环：
1. **调度器** 启动生成器（在独立 worktree 中工作）
2. **生成器** 基于历史反馈修改目标文件
3. **调度器** 启动评估器（看到生成器的改动）
4. **评估器** 以严格高标准独立评估改动质量
5. **调度器** 根据评估结果决定 **keep**（合并）或 **discard**（丢弃，零风险）
6. 记录结果，进入下一轮

## 核心特性

- **三角分工** — 生成器不知道评估标准，评估器不知道生成意图，调度器不直接改文件。职责分离确保客观性。
- **Git Worktree 隔离** — 每次改动在独立 worktree 中进行。discard = 不合并，不需要 `git reset`，零回滚风险。
- **完全自主** — 启动后无需人工干预。循环运行期间严格禁止向用户提问（绝对禁令）。
- **中断恢复** — 通过 `state.json` 精确跟踪每轮状态（5 个 phase），任何阶段中断都能从断点恢复。
- **严格评估** — 评估器以高标准执行，问题分级（Critical/Major/Minor），标准只升不降。
- **智能停止** — 支持轮数上限、连续失败/跳过/丢弃阈值，不会无意义地空转。

## 安装 Claude Code Skill

安装后可以在任何项目中直接使用 `/autoloop` 命令一键启动。

### 安装

将以下 prompt 粘贴到 Claude Code 中执行：

```
请帮我安装 autoloop skill。执行以下步骤：
1. 运行 git clone https://github.com/wujian/autoloop.git /tmp/autoloop-install
2. 运行 mkdir -p ~/.claude/skills/autoloop/templates
3. 运行 cp /tmp/autoloop-install/skill/SKILL.md ~/.claude/skills/autoloop/SKILL.md
4. 运行 cp /tmp/autoloop-install/templates/* ~/.claude/skills/autoloop/templates/
5. 运行 rm -rf /tmp/autoloop-install
6. 验证安装：运行 ls ~/.claude/skills/autoloop/SKILL.md ~/.claude/skills/autoloop/templates/ 确认文件存在
安装完成后告诉我结果。
```

安装完成后，重启 Claude Code，在任意 Git 仓库中输入 `/autoloop` 即可使用。

### 卸载

将以下 prompt 粘贴到 Claude Code 中执行：

```
请帮我卸载 autoloop skill。执行以下步骤：
1. 运行 rm -rf ~/.claude/skills/autoloop
2. 验证卸载：运行 ls ~/.claude/skills/autoloop 2>&1 确认目录已删除
卸载完成后告诉我结果。
```

重启 Claude Code 即生效。

### 验证安装

在 Claude Code 中输入 `/autoloop`。如果看到交互式引导（询问角色、任务目标等），说明安装成功。

## Quick Start

有两种使用方式：**通过 Skill**（推荐）或**手动复制模板**。

### 方式一：通过 Skill（推荐）

1. 按上方说明安装 skill
2. 在你的项目目录（Git 仓库）中打开 Claude Code
3. 输入 `/autoloop`，按引导完成配置
4. 循环自动启动

### 方式二：手动复制模板

#### 1. 准备工作目录

在你的项目根目录（Git 仓库）中，复制模板文件：

```bash
cp autoloop/templates/program.md    ./
cp autoloop/templates/generator.md  ./
cp autoloop/templates/evaluator.md  ./
```

#### 2. 编写 config.md

参考 `templates/config.md` 或 `examples/` 中的示例，创建你自己的配置：

```yaml
## Identity
role: "你的角色"
task: "你的任务目标"

## Workspace
target_files:
  - 要修改的文件
readonly_files:
  - 只读参考文件

## Evaluation
evaluation_methods:
  - command: "评估命令"
    description: "这个命令检查什么"
  - read_files: true
```

#### 3. 启动

在 Claude Code 中：

```
读取 program.md，开始 Setup
```

调度器会引导你完成初始化，然后自动开始循环。

## 配置参考

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `role` | AI 扮演的角色 | — |
| `task` | 一句话任务目标 | — |
| `target_files` | 可修改的文件列表 | — |
| `readonly_files` | 只读参考文件 | — |
| `generator_strategy` | `one_change` / `multi_change` / `exploratory` | `one_change` |
| `evaluator_depth` | `quantitative` / `qualitative` / `mixed` | `mixed` |
| `evaluation_methods` | 评估命令和方式 | — |
| `max_rounds` | 最大循环轮数 | `100` |
| `max_consecutive_fail` | 连续失败停止阈值 | `3` |
| `max_consecutive_discard` | 连续丢弃停止阈值 | `10` |
| `max_consecutive_skip` | 连续跳过停止阈值 | `5` |
| `report_interval` | 每 N 轮输出进度摘要 | `5` |
| `timeout_minutes` | 单轮超时（分钟） | `10` |
| `history_window` | 传递给 subagent 的历史轮数 | `40` |

## 使用场景

| 场景 | role | generator_strategy | evaluator_depth |
|------|------|--------------------|-----------------|
| ML 实验（超参/架构搜索） | 自主 ML 研究员 | `one_change` | `quantitative` |
| API 文档优化 | 文档优化师 | `one_change` | `mixed` |
| Bug 修复 | Bug 修复工程师 | `multi_change` | `quantitative` |
| 代码重构 | 重构工程师 | `multi_change` | `qualitative` |
| 架构探索 | 架构师 | `exploratory` | `qualitative` |

更多示例见 [`examples/`](examples/)。

## 项目结构

```
autoloop/
  templates/           ← 核心模板（复制到项目目录使用）
    program.md         ← 调度器指令（主循环逻辑）
    generator.md       ← 生成器指令
    evaluator.md       ← 评估器指令
    config.md          ← 配置模板
  skill/               ← Claude Code skill（可选）
    SKILL.md           ← /autoloop 命令入口
  examples/            ← 使用示例
    ml-research.md     ← ML 实验配置
    doc-optimization.md← 文档优化配置
```

## 运行时产物

循环启动后自动生成（已加入 `.gitignore`）：

```
.run/
  dispatcher/
    state.json         ← 当前状态（中断恢复）
    results.jsonl      ← 所有轮次结果
  generator/
    task.md            ← 自动生成的任务 prompt
    gen/
      summary.md       ← 生成历史摘要
      round_001.md     ← 各轮详细记录
  evaluator/
    task.md            ← 自动生成的评估标准
    eval/
      summary.md       ← 评估历史摘要
      round_001.md     ← 各轮详细评估
```

## 与 autoresearch 的关系

autoloop 的灵感直接来自 Andrej Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch) 项目。autoresearch 展示了一个优雅的理念：让 AI agent 像研究员一样自主实验——修改代码、运行训练、评估结果、保留或丢弃，循环往复。

autoloop 将这个理念从"ML 实验"泛化到"任何迭代改进任务"，同时引入了三角分工架构来提升评估的客观性和操作的安全性。

> *"Research is now entirely the domain of autonomous swarms of AI agents running across compute cluster megastructures in the skies."*
> — Andrej Karpathy

感谢 Karpathy 和 autoresearch 社区的开创性工作。

## License

[MIT](LICENSE)
