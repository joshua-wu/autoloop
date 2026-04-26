# autoloop

**Turn any task into an autonomous improvement loop. Start before bed, wake up to results.**

English | [中文](README.md)

autoloop is a general-purpose, three-agent autonomous loop framework inspired by Karpathy's [autoresearch](https://github.com/karpathy/autoresearch). It abstracts the "modify-evaluate-decide" cycle into reusable templates, enabling AI agents to autonomously, continuously, and safely improve any target file — whether it's training code, API docs, or config files.

## How It Works

```
                    ┌─────────────┐
                    │  Dispatcher  │
                    │  decide &    │
                    │  record      │
                    └──┬───────┬──┘
               keep/   │       │   eval report
              discard  │       │
                ┌──────┴──┐ ┌──┴──────┐
                │Generator│ │Evaluator│
                │ modify   │ │ assess  │
                │ files    │ │ quality │
                └─────────┘ └─────────┘
                      ↑           ↑
                      │  worktree  │
                      │ isolation  │
                      └─────┬─────┘
                            │
                     ┌──────┴──────┐
                     │ Target Files │
                     │ (your code)  │
                     └─────────────┘
```

Each round:
1. **Dispatcher** spawns the Generator (working in an isolated worktree)
2. **Generator** modifies target files based on historical feedback
3. **Dispatcher** spawns the Evaluator (sees the Generator's changes)
4. **Evaluator** independently assesses quality with strict, high standards
5. **Dispatcher** decides: **keep** (merge) or **discard** (drop — zero risk)
6. Record results, next round

## Key Features

- **Separation of concerns** — Generator doesn't know evaluation criteria. Evaluator doesn't know generation intent. Dispatcher doesn't touch files. Role isolation ensures objectivity.
- **Git worktree isolation** — Every change happens in a temporary worktree. Discard = don't merge. No `git reset`, zero rollback risk.
- **Fully autonomous** — No human intervention after startup. The loop strictly forbids asking the user anything during execution.
- **Crash recovery** — `state.json` tracks 5 phases per round. Interruptions at any point resume from exactly where they left off.
- **Strict evaluation** — Evaluator applies high standards with severity grading (Critical/Major/Minor). Standards only go up, never down.
- **Smart stopping** — Configurable limits: max rounds, consecutive fail/skip/discard thresholds. No pointless spinning.

## Install as Claude Code Skill

Once installed, use `/autoloop` in any project to launch with one command.

### Install

Paste the following prompt into Claude Code:

```
Please install the autoloop skill. Execute these steps:
1. Run git clone https://github.com/joshua-wu/autoloop.git /tmp/autoloop-install
2. Run mkdir -p ~/.claude/skills/autoloop/templates
3. Run cp /tmp/autoloop-install/skill/SKILL.md ~/.claude/skills/autoloop/SKILL.md
4. Run cp /tmp/autoloop-install/templates/* ~/.claude/skills/autoloop/templates/
5. Run rm -rf /tmp/autoloop-install
6. Verify: run ls ~/.claude/skills/autoloop/SKILL.md ~/.claude/skills/autoloop/templates/ to confirm files exist
Tell me the result when done.
```

After installation, restart Claude Code. Type `/autoloop` in any Git repository to use it.

### Uninstall

Paste the following prompt into Claude Code:

```
Please uninstall the autoloop skill. Execute these steps:
1. Run rm -rf ~/.claude/skills/autoloop
2. Verify: run ls ~/.claude/skills/autoloop 2>&1 to confirm the directory is deleted
Tell me the result when done.
```

Restart Claude Code to take effect.

### Verify Installation

Type `/autoloop` in Claude Code. If you see an interactive guide (asking about role, task objective, etc.), the installation is successful.

## Quick Start

Two ways to use autoloop: **via Skill** (recommended) or **manual template copy**.

### Option 1: Via Skill (Recommended)

1. Install the skill as described above
2. Open Claude Code in your project directory (Git repository)
3. Type `/autoloop` and follow the interactive guide
4. The loop starts automatically

### Option 2: Manual Template Copy

#### 1. Prepare your workspace

In your project root (a Git repository), copy the template files:

```bash
cp autoloop/templates/program.md    ./
cp autoloop/templates/generator.md  ./
cp autoloop/templates/evaluator.md  ./
```

#### 2. Write config.md

Use `templates/config.md` or `examples/` as a reference to create your config:

```yaml
## Identity
role: "Your role"
task: "Your task objective"

## Workspace
target_files:
  - files/to/modify
readonly_files:
  - reference/files

## Evaluation
evaluation_methods:
  - command: "your eval command"
    description: "what this checks"
  - read_files: true
```

#### 3. Launch

In Claude Code:

```
Read program.md and start Setup
```

The Dispatcher will guide you through initialization, then the loop runs automatically.

## Configuration Reference

| Config | Description | Default |
|--------|-------------|---------|
| `role` | Role the AI plays | — |
| `task` | One-line task objective | — |
| `target_files` | Files the Generator can modify | — |
| `readonly_files` | Read-only reference files | — |
| `generator_strategy` | `one_change` / `multi_change` / `exploratory` | `one_change` |
| `evaluator_depth` | `quantitative` / `qualitative` / `mixed` | `mixed` |
| `evaluation_methods` | Eval commands and methods | — |
| `max_rounds` | Maximum loop iterations | `100` |
| `max_consecutive_fail` | Consecutive failures before stop | `3` |
| `max_consecutive_discard` | Consecutive discards before stop | `10` |
| `max_consecutive_skip` | Consecutive skips before stop | `5` |
| `report_interval` | Progress summary every N rounds | `5` |
| `timeout_minutes` | Per-round timeout (minutes) | `10` |
| `history_window` | History rounds passed to subagents | `40` |

## Use Cases

| Scenario | role | generator_strategy | evaluator_depth |
|----------|------|--------------------|-----------------|
| ML experiments | ML Researcher | `one_change` | `quantitative` |
| API doc improvement | Doc Optimizer | `one_change` | `mixed` |
| Bug fixing | Bug Fix Engineer | `multi_change` | `quantitative` |
| Code refactoring | Refactoring Engineer | `multi_change` | `qualitative` |
| Architecture exploration | Architect | `exploratory` | `qualitative` |

See [`examples/`](examples/) for detailed configs.

## Project Structure

```
autoloop/
  templates/           ← Core templates (copy to your project)
    program.md         ← Dispatcher instructions (main loop)
    generator.md       ← Generator instructions
    evaluator.md       ← Evaluator instructions
    config.md          ← Config template
  skill/               ← Claude Code skill (optional)
    SKILL.md           ← /autoloop command entry point
  examples/            ← Usage examples
    ml-research.md     ← ML experiment config
    doc-optimization.md← Doc optimization config
```

## Runtime Artifacts

Auto-generated when the loop starts (gitignored):

```
.run/
  dispatcher/
    state.json         ← Current state (crash recovery)
    results.jsonl      ← All round results
  generator/
    task.md            ← Auto-generated task prompt
    gen/
      summary.md       ← Generation history summary
      round_001.md     ← Per-round detailed records
  evaluator/
    task.md            ← Auto-generated eval criteria
    eval/
      summary.md       ← Evaluation history summary
      round_001.md     ← Per-round detailed assessments
```

## Acknowledgments

autoloop is directly inspired by Andrej Karpathy's [autoresearch](https://github.com/karpathy/autoresearch). autoresearch demonstrated an elegant idea: let AI agents experiment like researchers — modify code, train, evaluate, keep or discard, repeat.

autoloop generalizes this concept from "ML experiments" to "any iterative improvement task", while introducing a three-agent architecture for better evaluation objectivity and operational safety.

> *"Research is now entirely the domain of autonomous swarms of AI agents running across compute cluster megastructures in the skies."*
> — Andrej Karpathy

Thanks to Karpathy and the autoresearch community for the pioneering work.

## License

[MIT](LICENSE)
