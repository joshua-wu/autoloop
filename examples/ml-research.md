# ML Research — 复现 autoresearch

这个配置将 autoloop 用于自主 ML 实验，等效于 [autoresearch](https://github.com/karpathy/autoresearch) 的功能。

## config.md

```yaml
# Task Configuration

## Identity
role: "自主 ML 研究员"
task: "通过修改 train.py 降低 val_bpb（验证集 bits per byte），在固定 5 分钟训练时间内找到最优模型"

## Workspace
branch_prefix: "autoloop"
target_files:
  - train.py
readonly_files:
  - prepare.py
  - pyproject.toml

## Commit Prefixes
dispatcher_prefix: "[dispatcher]"
generator_prefix: "[experiment]"
evaluator_prefix: "[evaluator]"

## Generator Strategy
generator_strategy: "one_change"

## Evaluator Depth
evaluator_depth: "quantitative"

## Evaluation
evaluation_methods:
  - command: "uv run train.py > run.log 2>&1 && grep '^val_bpb:\\|^peak_vram_mb:\\|^mfu_percent:\\|^num_params_M:' run.log"
    description: "运行 5 分钟训练并提取核心指标"
  - command: "tail -n 50 run.log"
    description: "检查训练是否崩溃"
  - compare_source: true

## History
history_window: 40
summary_max_bytes: 500
result_max_bytes: 200

## Stop Conditions
max_rounds: 100
max_consecutive_skip: 5
max_consecutive_discard: 10
max_consecutive_fail: 3

## Progress Report
report_interval: 5

## Constraints
timeout_minutes: 10
max_fix_attempts: 3

## Context Files
context:
  - README.md
  - prepare.py
```

## 与原版 autoresearch 的对比

| | autoresearch | autoloop |
|---|---|---|
| 架构 | 单 agent 改代码+跑实验+判断 | 三 agent 分工（Generator 改代码、Evaluator 跑实验、Dispatcher 判断） |
| 回滚 | `git reset` | 不 merge worktree（零风险） |
| 结果记录 | `results.tsv` | `results.jsonl` |
| 评估客观性 | 自己评自己 | 评估器独立客观，不知道生成器的意图 |
| 历史管理 | 同一 context 累积 | 每轮干净 context + summary.md 传递历史 |
| 中断恢复 | 无 | state.json + phase 跟踪 |

## 使用方式

1. 将 autoresearch 仓库克隆到本地
2. 将上述 config.md 保存到仓库根目录
3. 将 autoloop 的 `templates/` 中的 `program.md`、`generator.md`、`evaluator.md` 复制到仓库根目录
4. 启动 Claude Code，读取 `program.md` 开始 Setup
