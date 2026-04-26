# 文档优化

这个配置使用 autoloop 持续改善 API 文档的质量。

## config.md

```yaml
# Task Configuration

## Identity
role: "文档优化师"
task: "持续改善 API 文档的准确性、完整性和可读性"

## Workspace
branch_prefix: "autoloop"
target_files:
  - docs/api.md
  - docs/getting-started.md
readonly_files:
  - src/api/
  - README.md

## Commit Prefixes
dispatcher_prefix: "[dispatcher]"
generator_prefix: "[generator]"
evaluator_prefix: "[evaluator]"

## Generator Strategy
generator_strategy: "one_change"

## Evaluator Depth
evaluator_depth: "mixed"

## Evaluation
evaluation_methods:
  - command: "vale docs/"
    description: "文档风格检查"
  - command: "grep -c 'TODO' docs/"
    description: "未完成标记数量"
  - read_files: true
  - compare_source: true

## History
history_window: 40
summary_max_bytes: 500
result_max_bytes: 200

## Stop Conditions
max_rounds: 50
max_consecutive_skip: 3
max_consecutive_discard: 8
max_consecutive_fail: 3

## Progress Report
report_interval: 5

## Constraints
timeout_minutes: 10
max_fix_attempts: 3

## Context Files
context:
  - README.md
  - CONTRIBUTING.md
```

## 适用场景

- API 文档与源码不同步
- 文档缺少请求/响应示例
- 新功能已上线但文档未更新
- 文档结构混乱需要重组

## 使用方式

1. 在项目根目录保存上述 config.md
2. 复制 autoloop 的 `program.md`、`generator.md`、`evaluator.md` 到项目根目录
3. 启动 Claude Code，读取 `program.md` 开始 Setup
