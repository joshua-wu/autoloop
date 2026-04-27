# Task Configuration

## Identity
role: "文档优化师"
task: "持续改善 API 文档的准确性和可读性"

## Workspace
branch_prefix: "autoloop"
target_files:
  - docs/api.md
  - docs/getting-started.md
readonly_files:
  - src/api/
  - README.md

## Commit Prefixes
# 每个角色的 git commit 消息前缀，用于区分提交来源
dispatcher_prefix: "[dispatcher]"
generator_prefix: "[generator]"
evaluator_prefix: "[evaluator]"

## Generator Strategy
# one_change    — 每轮只做一个聚焦的改动（适合文档优化、超参调优）
# multi_change  — 每轮可做多个相关改动（适合 bug 修复、重构）
# exploratory   — 每轮做一个大胆的探索性改动（适合架构实验）
generator_strategy: "one_change"

## Evaluator Depth
# quantitative  — 以量化指标为主（命令输出、数值对比）
# qualitative   — 以定性分析为主（内容审查、可读性评估）
# mixed         — 量化 + 定性结合
evaluator_depth: "mixed"

## Evaluation
evaluation_methods:
  - command: "vale docs/"
    description: "文档风格检查"
  - command: "grep -c 'TODO' docs/"
    description: "未完成标记数量"
  - read_files: true
  - compare_source: true

## Critical Rules (质量底线)
# 违反任何一条即标记为 Critical，强烈倾向 discard
critical_rules:
  - "不得引入已通过测试的回归"
  - "不得删除已有功能"

## History
history_window: 40              # 只传最近 N 轮的 summary 给 subagent
summary_max_bytes: 500          # 每轮 summary 摘要的上限（字节）
result_max_bytes: 200           # results.jsonl 每行 description 字段的上限（字节）

## Stop Conditions
max_rounds: 100                 # 达到此轮数后停止循环
max_consecutive_skip: 5         # 连续 skip（无改动）次数达到此值后停止
max_consecutive_discard: 10     # 连续 discard 次数达到此值后停止
max_consecutive_fail: 3         # 连续 fail（重试耗尽）次数达到此值后停止

## Progress Report
report_interval: 5              # 每 N 轮输出一次进度摘要（纯输出，不暂停，不等待用户回复）

## Constraints
timeout_minutes: 10
max_fix_attempts: 3

## Context Files
context:
  - README.md
  - CONTRIBUTING.md
