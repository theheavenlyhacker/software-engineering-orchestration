# software-engineering-orchestration

Agent skills that drive GitHub issues from `agent-ready` to merged PR using supervised [Orca](https://github.com/stablyai/orca) workers: implement, open PR, 4-lens swarm review, fix loop, CI, merge, next issue.

Design background: [orca-ai-agent-orchestration.md](orca-ai-agent-orchestration.md).

## Prerequisites

**This only works on a GitHub repository that has GitHub issues.** The coordinator's work queue is open issues labeled `agent-ready`. No issues, nothing to do: it reports `NO_ELIGIBLE_OPEN_ISSUES` and ends. Issues must be enabled on the repo (Settings > Features > Issues) and written clearly enough to implement (acceptance criteria help).

Also required:

- [Orca](https://github.com/stablyai/orca) with its `orchestration` skill.
- `gh` installed and authenticated (`gh auth status`) with permission to label, open PRs, and merge.
- [mattpocock/skills](https://github.com/mattpocock/skills) installed for each worker agent: `npx skills@latest add mattpocock/skills -a <agent>`, then run `/setup-matt-pocock-skills` once per repo.
- Labels: `agent-ready`, `agent-in-progress`, `agent-review`, `agent-blocked`, `agent-needs-human`, `agent-completed` (the coordinator creates missing ones).
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

Label the issues you want handled `agent-ready`, then ask your agent:

> /issue-orchestration process the agent-ready issues

## How the loop works

The coordinator is small and deterministic: **Select, Dispatch, Wait, Evaluate, Route, Merge, Repeat.** Orca decides who works and when; the installed engineering skills decide how.

```text
                 GitHub issues labeled agent-ready
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
        +-- next agent-ready issue (stop when none)
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
agent-ready -> agent-in-progress -> agent-review -> agent-completed
                     |                   |
                     +--> agent-blocked  +--> agent-needs-human
```

## Skills

- `issue-orchestration`: the issue-to-merge coordinator and its role prompts (implementer, pr-opener/merger, code-fixer, reviewer, plus optional issue-analyzer, ci-fixer, conflict-resolver).
- `code-review-swarm`: 4-lens parallel reviewer (standards, spec, security, QA). Also usable standalone on any PR or diff.

## Agents

Workers run as `codex`, `claude`, or `opencode` via Orca's `--agent`. Defaults: codex implements and fixes, claude reviews. Tell the coordinator to override per role (e.g. "use opencode as reviewer"). Reviewer must differ from implementer.

## Customize

Override any role per repository with `.orca/roles/<role>.md` (implementer, reviewer, code-fixer, merger, issue-analyzer, ci-fixer, conflict-resolver). Defaults live in `skills/issue-orchestration/roles/`.
