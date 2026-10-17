---
name: issue-orchestration
description: Drive GitHub issues labeled agent-ready through implement, independent review, PR, CI, conflict resolution, and merge using supervised Orca workers, then repeat until the queue is empty. Use when the user asks to orchestrate, process, or burn down GitHub issues with multiple agents.
---

# Issue Orchestration

Coordinator loop. Orca decides WHO works and WHEN. Installed engineering skills (tdd, code-review, pr, diagnosing-bugs, ...) decide HOW. Keep this coordinator small: Select, Dispatch, Wait, Evaluate, Route, Merge, Repeat.

## Prerequisites

1. Load the `orchestration` skill (Orca) first and follow its CLI-resolution and supervised-loop rules. Do not duplicate them here. Never substitute a non-Orca subagent tool.
2. `gh auth status` passes. Labels exist: `agent-ready`, `agent-in-progress`, `agent-review`, `agent-blocked`, `agent-needs-human`, `agent-completed` (create missing with `gh label create`).
3. Engineering skills from https://github.com/mattpocock/skills are installed for every worker agent (`npx skills@latest add mattpocock/skills -a <agent>`; Codex/Claude plugins also work, pick one method per agent). Run `/setup-matt-pocock-skills` once per repo (GitHub tracker, labels, docs location). Roles use: implement, tdd, codebase-design, diagnosing-bugs, code-review, pr, triage, research, domain-modeling. If one is missing, the worker proceeds without it and says so in its report.

## Roles

Role prompts live in `roles/` beside this file. Resolution order per role: `<repo>/.orca/roles/<role>.md` (repo override) then this skill's `roles/<role>.md`.
Workers do not see this skill directory: **inline the role file text into every `--spec`**, plus issue number, branch, and prior findings.

| Role | Default agent | Verdicts |
|---|---|---|
| implementer | codex | `IMPLEMENTATION_COMPLETE` |
| reviewer | claude | `APPROVED` / `CHANGES_REQUESTED` |
| issue-analyzer (optional) | claude | `READY_FOR_IMPLEMENTATION` / `BLOCKED` |
| ci-fixer (optional) | codex | `CI_FIX_COMPLETE` / `ESCALATE` |
| conflict-resolver (optional) | codex | `RESOLVED` / `ESCALATE` |
| merger | codex | PR URL, merged |

Reviewer agent MUST differ from the implementer's agent. Implementer never approves or merges.
Start with MVP: implementer, reviewer, merger. Add optional roles only when the user asks or the failure mode appears.
Agents are defaults; honor user overrides. Supported `--agent` values: `codex`, `claude`, `opencode` (or any other agent Orca can launch). Use `opencode` for any role the user assigns it, e.g. a cheaper implementer or a third-party reviewer; the implementer/reviewer difference rule still applies.

## Loop

```text
run-create once. Then per issue:
select -> label agent-in-progress -> [analyze] -> implement -> review
  CHANGES_REQUESTED -> implement fixes -> review   (max 3 cycles)
  APPROVED -> label agent-review -> create PR ("Closes #N", merger role)
  CI fail -> ci-fixer -> review again -> CI           (max 2)
  conflict -> conflict-resolver -> review -> CI       (max 2)
  all gates green -> gh pr merge --squash --delete-branch
  verify `gh issue view N --json state` == CLOSED -> label agent-completed
next issue; stop when none eligible.
```

- Select: `gh issue list --state open --label agent-ready --json number,title,body,labels,url`, lowest number first. One issue at a time (no parallel unless user asks).
- Worktree per issue, branch `agent/issue-<N>-<slug>`. Implementer, fixer, and resolver share it; reviewer reads it.
- Dispatch with `worker-start --spec ... --worktree ... --agent ...`; wait with `check --wait --types worker_done,escalation,question`. Parse the verdict token from the worker_done summary. No verdict or unknown token counts as failed attempt.
- Any fix that changes the diff (CI fix, conflict resolution) goes back to the reviewer. Never merge on a stale approval.
- Release each settled worker per the orchestration skill's completion accounting.

## Limits

Review cycles 3, implementation attempts 3, CI fixes 2, conflict attempts 2, analysis 2. Exceeded: label `agent-needs-human`, comment with the reason, skip to next issue. `BLOCKED`: label `agent-blocked`, comment, skip.

## Safety floor

- Only `agent-ready` issues. Never touch others.
- Never bypass branch protection, force-push protected branches, merge with red or pending required checks, change secrets, or delete unrelated branches.
- Infra, auth, or migration changes: stop at `agent-needs-human` before merge.
- If the repo sets `.orca/policies/merge.md` with `human-approval: required`, stop after CI green and ask the user before merging.
- Ask the user before the first merge of a run unless they already authorized unattended merging.

## Report

End with per issue: outcome, PR URL, review cycles used, CI result, blocker if any.
