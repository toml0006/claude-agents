# claude-agents

My [Claude Code subagent](https://docs.claude.com/en/docs/claude-code/sub-agents) definitions. They live in `~/.claude/agents/`.

These are shared as a point of reference, not as drop-in agents. They are written for my own setup. `orchestrator` in particular hands work to my dev boxes (`artoo`, `threepio`) through a `devbox-dispatch` skill that is not included here.

| Agent | What it does |
|---|---|
| `orchestrator` | Runs a multi-PR or multi-slice effort: plans, hands each task to an `implementer`, and tracks results from short reports. Launch with `claude --agent orchestrator`. |
| `implementer` | Implements one planned slice, PR, rebase or CI fix in the worktree its brief names, then returns a short report. |
| `codebase-memory-scout` | Tier 1: fast, provisional lookup in the [codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) graph. |
| `codebase-memory` | Tier 2 (default): task-directed graph verification with coverage checks and source fallback. |
| `codebase-memory-auditor` | Tier 3: bounded-scope full audit of the graph, disclosing every limitation. |

The `codebase-memory*` agents need the `codebase-memory-mcp` server and its `codebase-memory` skill.

## Install

```sh
git clone https://github.com/toml0006/claude-agents.git ~/.claude/agents
```
