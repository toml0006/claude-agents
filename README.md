# claude-agents

My [Claude Code subagent](https://docs.claude.com/en/docs/claude-code/sub-agents) definitions, plus the `devbox-dispatch` skill the orchestrator uses to hand work to my dev boxes.

These are shared as a point of reference, not as drop-in agents. They are written for my own setup: `orchestrator` hands work to two dev boxes (`artoo`, `threepio`) through `devbox-dispatch`, and that skill assumes those boxes exist and are reachable over SSH.

## Agents

| Agent | What it does |
|---|---|
| `orchestrator` | Runs a multi-PR or multi-slice effort: plans, hands each task to an `implementer`, and tracks results from short reports. Launch with `claude --agent orchestrator`. |
| `implementer` | Implements one planned slice, PR, rebase or CI fix in the worktree its brief names, then returns a short report. |
| `codebase-memory-scout` | Tier 1: fast, provisional lookup in the [codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) graph. |
| `codebase-memory` | Tier 2 (default): task-directed graph verification with coverage checks and source fallback. |
| `codebase-memory-auditor` | Tier 3: bounded-scope full audit of the graph, disclosing every limitation. |

The `codebase-memory*` agents need the `codebase-memory-mcp` server and its `codebase-memory` skill.

## Skills

| Skill | What it does |
|---|---|
| [`devbox-dispatch`](skills/devbox-dispatch/SKILL.md) | Writes a brief, starts a headless Codex or Claude Code run in a fresh git worktree on a dev box, and collects the report and diff later. All of it goes through one script, [`scripts/devbox`](skills/devbox-dispatch/scripts/devbox). |

## Install

Claude Code reads every `.md` file under `~/.claude/agents`, including subfolders, so link only the `agents` folder there:

```sh
git clone https://github.com/toml0006/claude-agents.git ~/dev/claude-agents
ln -s ~/dev/claude-agents/agents ~/.claude/agents
ln -s ~/dev/claude-agents/skills/devbox-dispatch ~/.claude/skills/devbox-dispatch
```
