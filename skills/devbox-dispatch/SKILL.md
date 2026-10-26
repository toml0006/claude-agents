---
name: devbox-dispatch
description: "Use when Jackson wants coding work sent to, handed off to, farmed out to, or run on one of his Proxmox dev boxes (artoo, threepio, 'the coding agent nodes', 'the fleet', 'a devbox') instead of in this session, or asks what is running there or whether a dispatched job finished - 'send this to artoo', 'hand these three fixes to the boxes', 'run this on threepio overnight', 'what are the boxes doing', 'did the handoff finish'."
argument-hint: "send <task> | status | list | result <job>"
---

# Send work to a dev box

`artoo` and `threepio` are two fenced VMs that run agent CLIs as the user `jackson` (no sudo),
reached with `ssh artoo` / `ssh threepio`. A job is: a written **brief**, a fresh **git
worktree** of the repo on the box, and one headless agent run that works in it, detached, until
it writes a **final report**. This session writes the brief, starts the job, and later reads the
result. The box agent does the work.

Everything goes through one script, `devbox`, on the Mac's PATH (`~/.local/bin/devbox`, a link
to `scripts/devbox` in this skill). If the command is missing, run
`bash ~/.claude/skills/devbox-dispatch/scripts/devbox` with the same arguments.

| Command | What it does |
|---|---|
| `devbox status` | load, free memory, running jobs and cloned repos per box |
| `devbox send [opts] brief.md` | create the worktree and start the agent; returns at once |
| `devbox list` | every job on both boxes with its state |
| `devbox wait <job> [--timeout s]` | block until the job ends, then print the result |
| `devbox result <job>` | state, final report, commits and diff stat |
| `devbox log <job> [-f] [-n N]` | the agent's log |
| `devbox exec <job> -- <cmd>` | run a command in the job's worktree |
| `devbox stop <job>` | kill a running job (worktree and branch are kept) |

`send` options: `--name` (job and worktree name, default the brief's file name), `--repo`
(default: the origin of the checkout you run it in), `--box`, `--agent codex|claude`, `--base`,
`--branch` (default `agent/<name>`), `--detach`, `--setup '<cmd>'`, `--effort`. Run
`devbox --help` for the full text.

## 1. Write the brief

The box agent starts with **nothing from this conversation**. The brief is its whole context, so
write it as a file in your scratch directory with these parts, in this order:

1. **Title**: one line naming the outcome.
2. **Why**: who needs this and what it is for, in two or three sentences, plus what the repo is
   (stack, the apps that matter). A reader who has never seen the project should understand.
3. **Task**: what to change or produce, with the file paths, symbols, issue text, error
   messages and reproduction steps you already found. Paste them; do not make the agent
   rediscover them. State what done looks like and which test or command proves it.
4. **Ground rules**: do not `git commit` (for `--agent claude`: commit on the current branch
   with a conventional message); never push, open a pull request or deploy; never touch staging
   or production; delete temp files. Add the task's own limits (files not to touch, read-only).
5. **Final report**: exactly what to return: what changed, the commands run and their real
   output, what was not verified, anything surprising, and a proposed commit message.

One job is one reviewable change. Split unrelated work into separate briefs and send them all;
they run in parallel, each in its own worktree.

## 2. Send it

```bash
devbox status                      # only when choosing a box by hand
devbox send --setup 'pnpm install --frozen-lockfile --prefer-offline' /path/to/fix-totals.md
```

- **Box.** Leave `--box` off: the script picks a box that already has the repo, then the one
  with fewer agents running. If neither has the repo it is cloned from this checkout's origin.
- **Base.** The worktree starts from the repo's default branch on origin, freshly fetched. Work
  that only exists locally is not on the box: push the branch first and pass
  `--base origin/<branch>`.
- **`--setup`** runs in the worktree before the agent, with network. Put dependency installs
  here: the `codex` sandbox has no network, so an install inside the job fails.
- **Agent.** `codex` (default) is sandboxed to the worktree with no network: it can edit and
  run tests, and cannot commit (Codex keeps `.git` read-only) or push. `claude` runs
  unsandboxed with network and commits its own work (only `git push` is denied); use it when
  the task needs the network or MCP tools.
- **`--detach`** for jobs that produce a report rather than a branch.

## 3. Collect the result

Start the wait in the background and carry on; do not sit in a foreground wait or poll by hand:

```bash
devbox wait fix-totals        # Bash with run_in_background: true
```

When it returns, read the final report, then check it against the worktree: `result` shows the
commits, diff stat and uncommitted files, and `exec <job> -- git diff` shows the change. The
report is the agent's own account; the diff is the evidence. Report both to Jackson, and say
which claims you checked.

A `codex` job leaves its work uncommitted. Once the diff is right, commit it from here:

```bash
devbox exec fix-totals -- git add -A
devbox exec fix-totals -- git commit -m "fix(checkout): recompute totals after address change"
```

To publish, only when Jackson has asked for a pull request:

```bash
devbox exec fix-totals -- git push -u origin agent/fix-totals
devbox exec fix-totals -- gh pr create --draft --fill
```

`gh` on the boxes maps each repository's organisation to the right GitHub account by itself.
Never `gh auth switch` there.

## States and failures

| `list` / `result` shows | Meaning | Next step |
|---|---|---|
| `running` | agent still working | `log <job> -f` to watch |
| `done rc=0` | agent exited cleanly | read the report **and** the diff |
| `done rc=98` | `--setup` failed, agent never ran | `result` prints the setup log; fix and resend under a new name |
| `done rc=143` | stopped with `stop` | worktree kept |
| `died` | process gone without an exit code (box restarted) | resend under a new name |
| `name ... is taken` | a job, brief or worktree with that name exists | choose another `--name` |
| `no box reachable` | the Mac is off the tailnet or the boxes are down | tell Jackson; do not fall back to doing the work locally unasked |

A job name is used once. To retry, send again as `<name>-2`; the first worktree stays for
comparison.

## Limits

- The boxes are fenced off the home LAN (Home Assistant on two ports is the exception). Work
  that needs the NAS, Proxmox or 1Password homelab secrets does not belong there.
- Nothing cleans up. Finished worktrees, branches and `~/handoffs` files stay until removed.
  Once a branch is merged or abandoned, and only when Jackson agrees:
  `ssh <box> 'git -C ~/<repo> worktree remove .claude/worktrees/<job>'`.
- Jobs started by hand on a box (`~/handoffs/start.sh`, plain `codex exec`) have no record under
  `~/handoffs/jobs` and do not appear in `list`; `status` still counts their processes.
