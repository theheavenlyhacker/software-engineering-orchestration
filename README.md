# software-engineering-orchestration

Agent skill that drives GitHub issues through implement, independent review, PR, CI, conflict resolution, and merge using supervised [Orca](https://github.com/stablyai/orca) workers.

Design background: [orca-ai-agent-orchestration.md](orca-ai-agent-orchestration.md).

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

Manual: copy `skills/issue-orchestration/` into `.claude/skills/` (repo) or `~/.claude/skills/` (global).

## Use

Requires Orca, `gh` (authenticated), Orca's `orchestration` skill, and [mattpocock/skills](https://github.com/mattpocock/skills) installed for each worker agent (`npx skills@latest add mattpocock/skills -a <agent>`, then run `/setup-matt-pocock-skills` once per repo). Label issues `agent-ready`, then ask your agent:

> /issue-orchestration process the agent-ready issues

## Skills

- `issue-orchestration`: the issue-to-merge coordinator.
- `code-review-swarm`: 4-lens parallel reviewer (standards, spec, security, QA). Also usable standalone on any PR or diff; it is the default reviewer step.

## Agents

Workers run as `codex`, `claude`, or `opencode` via Orca's `--agent`. Defaults: codex implements, claude reviews. Tell the coordinator to override per role (e.g. "use opencode as reviewer"). Reviewer must differ from implementer.

## Customize

Override any role per repository with `.orca/roles/<role>.md` (implementer, reviewer, issue-analyzer, ci-fixer, conflict-resolver, merger). Defaults live in `skills/issue-orchestration/roles/`.
