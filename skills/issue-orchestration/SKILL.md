---
name: issue-orchestration
description: Drive GitHub issues carrying the queue label (default ready-for-agent) through implement, independent review, PR, CI, conflict resolution, and merge using supervised Orca workers, then repeat until the queue is empty. Use when the user asks to orchestrate, process, or burn down GitHub issues with multiple agents.
---

# Issue Orchestration

Coordinator loop. Orca decides WHO works and WHEN. Installed engineering skills (tdd, code-review, pr, diagnosing-bugs, ...) decide HOW. Keep this coordinator small: Select, Dispatch, Wait, Evaluate, Route, Merge, Repeat.

## Prerequisites

1. Load the `orchestration` skill (Orca) first and follow its CLI-resolution and supervised-loop rules. Do not duplicate them here. Never substitute a non-Orca subagent tool.
2. `gh auth status` passes. Resolve the **queue label** once per run: the user's explicit choice, else the `ready-for-agent` mapping in `docs/agents/triage-labels.md` (written by `/setup-matt-pocock-skills`; the real string may differ from the canonical name), else `ready-for-agent` if that label exists in the repo, else `agent-ready`. Never relabel the user's issues to fit. Ensure these state labels exist (create missing with `gh label create`): `agent-in-progress`, `agent-review`, `agent-blocked`, `agent-needs-human`, `agent-completed` 
3. Engineering skills from https://github.com/mattpocock/skills are installed for every worker agent (`npx skills@latest add mattpocock/skills -a <agent>`; Codex/Claude plugins also work, pick one method per agent). Run `/setup-matt-pocock-skills` once per repo (GitHub tracker, labels, docs location). Roles use: implement, tdd, codebase-design, diagnosing-bugs, code-review, pr, triage, research, domain-modeling. If one is missing, the worker proceeds without it and says so in its report.

## Roles

Role prompts live in `roles/` beside this file. Resolution order per role: `<repo>/.orca/roles/<role>.md` (repo override) then this skill's `roles/<role>.md`.
Workers do not see this skill directory: **inline the role file text into every `--spec`**, plus issue number, branch, and prior findings.

| Role | Default agent | Verdicts |
|---|---|---|
| implementer | codex | `IMPLEMENTATION_COMPLETE` |
| pr-opener (merger role, open phase) | codex | PR URL |
| reviewer (`code-review-swarm`, runs on the open PR) | claude | `APPROVED` / `CHANGES_REQUESTED` |
| code-fixer | codex | `FIXES_COMPLETE` / `ESCALATE` |
| merger (merge phase) | codex | merged |
| issue-analyzer (optional) | claude | `READY_FOR_IMPLEMENTATION` / `BLOCKED` |
| ci-fixer (optional) | codex | `CI_FIX_COMPLETE` / `ESCALATE` |
| conflict-resolver (optional) | codex | `RESOLVED` / `ESCALATE` |

Reviewer agent MUST differ from the implementer's and code-fixer's agent. Implementer and code-fixer never approve or merge. Only the merger merges, and only after the coordinator confirms all gates.
Start with MVP: implementer, merger, reviewer, code-fixer. Add optional roles only when the user asks or the failure mode appears.
Agents are defaults; honor user overrides. Supported `--agent` values: `codex`, `claude`, `opencode`, `antigravity` (or any other agent id Orca lists). The reviewer-differs rule still applies to every assignment.

### Token optimization

Spend premium tokens only where judgment is hard. Route by role weight, using `--agent` and, where the agent supports it, `--model` / `--effort` (check `worker-start --help`; Antigravity takes `--model`; OpenCode picks models from its own config):

| Weight | Roles | Route |
|---|---|---|
| Light (mechanical) | pr-opener, merger, issue-analyzer, ci-fixer log triage | cheapest capable agent/model: `antigravity` fast model, or `opencode` with a small model |
| Medium | reviewer swarm lenses | `antigravity` fast tier by default; `--deep` only for risky diffs (auth, data, migrations) |
| Heavy | implementer, code-fixer, conflict-resolver | strongest coding agent, e.g. `codex` or `claude` |

Also: one worker per task (no idle terminals; release settled workers), put only the needed context in each `--spec` (issue body, findings, role text, not the whole history), and stop at retry limits instead of looping. Never downgrade the reviewer to the same agent as the implementer to save tokens.

## Loop

```text
run-create once. Then per issue:
select -> label agent-in-progress -> [analyze] -> implement (local checks green)
  -> OPEN PR first (pr-opener; "Closes #N"; label agent-review)
  -> code-review-swarm on the open PR (reviewer)
       APPROVED          -> wait CI
       CHANGES_REQUESTED -> code-fixer (findings in spec) pushes to PR branch
                            -> code-review-swarm again          (max 3 cycles)
  CI green + APPROVED on current head + no conflicts -> merger merges (squash, delete branch)
  CI red   -> ci-fixer -> swarm review again -> CI              (max 2)
  conflict -> conflict-resolver -> swarm review -> CI           (max 2)
verify `gh issue view N --json state` == CLOSED -> label agent-completed
next issue; stop when none eligible.
```

- Select: `gh issue list --state open --label <queue-label> --json number,title,body,labels,url`, lowest number first. One issue at a time (no parallel unless user asks).
- Worktree per issue, branch `agent/issue-<N>-<slug>`. Implementer, code-fixer, ci-fixer, and resolver share it; reviewer reads it.
- Review happens on the PR, never before it exists. Pass the reviewer the PR number so it diffs with `gh pr diff`.
- Dispatch with `worker-start --spec ... --worktree ... --agent ...`; wait with `check --wait --types worker_done,escalation,question`. Parse the verdict token from the worker_done summary. No verdict or unknown token counts as failed attempt.
- The merge gate is the latest review verdict on the PR's current head SHA (`gh pr view N --json headRefOid`). Any push (fix, CI fix, conflict resolution) invalidates earlier approval; re-run the swarm. Never merge on a stale approval.
- The reviewer posts its swarm summary as a PR comment. The coordinator holds the verdict and routes; do not trust a verdict that is not in the worker_done summary.
- Release each settled worker per the orchestration skill's completion accounting.

## Limits

Review/fix cycles 3, implementation attempts 3, CI fixes 2, conflict attempts 2, analysis 2. Exceeded: label `agent-needs-human`, comment with the reason, skip to next issue. `BLOCKED`: label `agent-blocked`, comment, skip.

## Safety floor

- Only issues with the queue label. Never touch others.
- Never bypass branch protection, force-push protected branches, merge with red or pending required checks, change secrets, or delete unrelated branches.
- Infra, auth, or migration changes: stop at `agent-needs-human` before merge.
- If the repo sets `.orca/policies/merge.md` with `human-approval: required`, stop after CI green and ask the user before merging.
- Ask the user before the first merge of a run unless they already authorized unattended merging.

## Report

End with per issue: outcome, PR URL, review cycles used, CI result, blocker if any.
