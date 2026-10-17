# software-engineering-orchestration

Agent skills that drive GitHub issues from the queue label (default `ready-for-agent`) to merged PR using supervised [Orca](https://github.com/stablyai/orca) workers: implement, open PR, 4-lens swarm review, fix loop, CI, merge, next issue.

Design background: [orca-ai-agent-orchestration.md](orca-ai-agent-orchestration.md).

## Prerequisites

**This only works on a GitHub repository that has GitHub issues.** The coordinator's work queue is open issues carrying the **queue label**: `ready-for-agent` by default (the canonical label from mattpocock's `triage`), resolved from `docs/agents/triage-labels.md` if your repo maps it differently, or any label you name ("use label X"). Your issues are never relabeled to fit. No issues, nothing to do: it reports `NO_ELIGIBLE_OPEN_ISSUES` and ends. Issues must be enabled on the repo (Settings > Features > Issues) and written clearly enough to implement (acceptance criteria help).

Also required:

- [Orca](https://github.com/stablyai/orca) with its `orchestration` skill.
- `gh` installed and authenticated (`gh auth status`) with permission to label, open PRs, and merge.
- [mattpocock/skills](https://github.com/mattpocock/skills) installed for each worker agent: `npx skills@latest add mattpocock/skills -a <agent>`, then run `/setup-matt-pocock-skills` once per repo.
- Labels: the queue label above plus state labels `agent-in-progress`, `agent-review`, `agent-blocked`, `agent-needs-human`, `agent-completed` (the coordinator creates missing ones).
- Recommended: branch protection with required CI checks. The coordinator never bypasses it.

## Install

Per repository (project-local):

```bash
npx skills@latest add theheavenlyhacker/software-engineering-orchestration -a claude-code
```

Global (all repositories):

```bash
npx skills@latest add theheavenlyhacker/software-engineering-orchestration -a claude-code -g
```

Replace `-a claude-code` with `codex`, `opencode`, etc., or omit it to choose interactively.

Manual: copy `skills/*` into `.claude/skills/` (repo) or `~/.claude/skills/` (global).

## Use

Label the issues you want handled `ready-for-agent` (or your queue label), then ask your agent:

> /issue-orchestration process the ready-for-agent issues

## How the loop works

The coordinator is small and deterministic: **Select, Dispatch, Wait, Evaluate, Route, Merge, Repeat.** Orca decides who works and when; the installed engineering skills decide how.

```text
                 GitHub issues with the queue label (ready-for-agent)
                              |
                              v
                   +--------------------+
                   |    COORDINATOR     |  one Orca Run, one issue at a time
                   +---------+----------+
                             |
        select issue, label agent-in-progress
                             |
                             v
                  [ issue-analyzer ]  optional
                     |          \
                READY            BLOCKED --> label agent-blocked --> next issue
                     v
                [ implementer ]  local lint/types/tests green
                             |
                             v
                [ pr-opener ]  open PR ("Closes #N"), label agent-review
                             |
                             v
        +------> [ reviewer: code-review-swarm on the open PR ] <------+
        |                    |                                         |
        |        APPROVED    |    CHANGES_REQUESTED                    |
        |           |        +-------------------> [ code-fixer ] ----+  push, re-review
        |           v                               (max 3 cycles, then agent-needs-human)
        |   wait for CI
        |      |        \
        |    green      red --> [ ci-fixer ] --(push)--> back to reviewer
        |      |
        |   conflicts? -- yes --> [ conflict-resolver ] --(push)--> back to reviewer
        |      | no
        |      v
        |   gates: APPROVED on current head SHA + CI green + no conflicts
        |      v
        |  [ merger ]  squash merge, delete branch
        |      v
        |  verify issue CLOSED, label agent-completed
        |      v
        +-- next queued issue (stop when none)
```

Rules baked into the loop:

- Review always happens on the open PR, never before it exists.
- Any new push (fix, CI fix, conflict resolution) invalidates earlier approval and triggers a fresh review.
- Reviewer agent differs from implementer and fixer agents. Only the merger merges.
- Retry limits (review/fix 3, CI fix 2, conflict 2) end in `agent-needs-human`, never an endless loop.

### Inside the review swarm

```text
open PR diff + linked issue + repo standards
        |
        +--> standards & minimalism --+
        +--> spec & scope ------------+--> review triage --> ACTIONABLE -> code-fixer
        +--> security & tenancy ------+    (dedupe,           NIT        -> non-blocking
        +--> QA & edge cases ---------+     classify)         AMBIGUOUS  -> ask / needs-human
```

### Issue label lifecycle

```text
ready-for-agent -> agent-in-progress -> agent-review -> agent-completed
                     |                   |
                     +--> agent-blocked  +--> agent-needs-human
```

## Skills

- `issue-orchestration`: the issue-to-merge coordinator and its role prompts (implementer, pr-opener/merger, code-fixer, reviewer, plus optional issue-analyzer, ci-fixer, conflict-resolver).
- `code-review-swarm`: 4-lens parallel reviewer (standards, spec, security, QA). Also usable standalone on any PR or diff.

## Agents

Workers run as `codex`, `claude`, `opencode`, or `antigravity` via Orca's `--agent` (any agent id Orca lists works). Defaults: codex implements and fixes, claude reviews. Tell the coordinator to override per role (e.g. "use antigravity as reviewer"). Reviewer must differ from implementer.

### Token optimization

Premium tokens go to hard judgment; mechanical work goes to cheap agents.

| Weight | Roles | Route |
|---|---|---|
| Light | pr-opener, merger, issue-analyzer | `antigravity` fast model or `opencode` small model |
| Medium | review swarm lenses | `antigravity` fast tier; `--deep` only for risky diffs |
| Heavy | implementer, code-fixer, conflict-resolver | `codex` or `claude` |

Example: "use antigravity for reviews and PR/merge, codex for implementation and fixes."

## Troubleshooting

- `Unknown flag --app` from `orca-ide` (Arch/Linux package): `/usr/bin/orca-ide` is an Electron launcher that injects `--app`. Use the `orca` shim in `~/.config/orca/linux-orca-cli-shim/orca` (on PATH inside Orca terminals) or `/usr/lib/orca-ide/bin/orca-ide`. Verify with `orca status --json`.
- Bare `orca` starting speech: that is the GNOME screen reader (`/usr/bin/orca`), not Orca ADE. Use the shim path instead.

## Customize

Override any role per repository with `.orca/roles/<role>.md` (implementer, reviewer, code-fixer, merger, issue-analyzer, ci-fixer, conflict-resolver). Defaults live in `skills/issue-orchestration/roles/`.
