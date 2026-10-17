---
name: code-review-swarm
description: Four-lens parallel code review (standards, spec, security, QA) with triage into ACTIONABLE, NIT, and AMBIGUOUS, for GitHub PRs and local diffs. Use when asked to review a PR, branch, or diff, or as the reviewer step of issue-orchestration.
---

# Code Review Swarm

Runs in the local agent session (no CI minutes). Spawns 4 parallel sub-agents, one per lens, then triages. Reviewer stance: senior engineer, objective, direct. No emojis anywhere; use plain markers PASS, WARN, FAIL, BLOCKER, NIT.

## Models

Default: fast/budget tier for all 4 lenses (quota-friendly). With `--deep`: Standards and Spec on the strongest reasoning tier, Security and QA on fast or standard tier. Pick the harness's own tier names (e.g. Claude Code haiku/sonnet/opus, Codex mini/standard/reasoning, Antigravity flash/inherit/pro).

## 1. Pin target and capture diff

- PR (`123`, `#123`, or URL): `gh pr view <n> --json number,title,body,headRefName,baseRefName,author,url` and `gh pr diff <n>`. Empty or closed: stop and report.
- Local ref: `git rev-parse <ref>`, `git diff <ref>...HEAD`, `git log <ref>..HEAD --oneline`.

## 2. Find spec and domain sources

Linked issue (`Closes #N` in PR body or commits): `gh issue view <n> --comments`. Else a path the user passed, else a PRD/ADR matching the branch name under `docs/` or `.scratch/`. None found: say "No formal spec available; reviewing against PR description and domain vocabulary" and continue.

## 3. Load standards

Read what exists: `CODING_STANDARDS.md`, `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md` (domain glossary, tenancy and RBAC rules), `docs/agents/`. Repo-specific rules below ("if the repo defines...") apply only when these files define them.

Smell baseline (Fowler, Refactoring ch.3), always applied: Mysterious Name, Duplicated Code, Feature Envy, Data Clumps, Primitive Obsession, Repeated Switches, Shotgun Surgery, Speculative Generality, Middle Man.

## 4. Spawn 4 reviewers in one parallel message

Every finding: exact file path and line range, quoted code, concrete failure or impact. Each reviewer under 400 words, plain text.

1. **standards-reviewer** (Standards & Minimalism): repo conventions and layering if defined; schema and annotation rules if defined; YAGNI, no gratuitous abstraction, stdlib and existing helpers over new code, comments explain why; smell baseline.
2. **spec-reviewer** (Spec & Scope): every requirement implemented? unrequested behavior or scope creep? terminology matches `CONTEXT.md`? Quote the spec line and the affected hunk or omission site.
3. **security-reviewer** (Security & Tenancy): auth guards on every route unless explicitly public; permission checks use valid keys; constant-time or dummy-hash credential comparison; tenant isolation scoped by the owning unit the repo defines (not the author) ; SQL parameterized; secrets, injection, SSRF, path traversal. Rate each High/Med/Low.
4. **qa-reviewer** (QA & Edge Cases): tests exist and follow the repo's test layout and runner; DB tests clean up; N+1 and unindexed lookups; races and concurrency windows; failure scenarios described step by step. Run the tests when feasible and report results.

## 5. Triage

Deduplicate overlapping findings across lenses into one thread, then classify:

- **ACTIONABLE**: clear, high-certainty bug, standards breach, or missing guard. Must have `path:Lstart-Lend`, the quoted code, and a replacement as a GitHub suggestion block.
- **NIT**: non-blocking style or naming. Needs file, line, quote.
- **AMBIGUOUS**: architectural tradeoff, conflicting requirements, design fork.

## 6. Interaction

**Standalone (human present):** present AMBIGUOUS items and ask for a decision on each. Offer for ACTIONABLE: (A) inline suggestion comments on the PR (default) or (B) check out the branch, commit the fixes, push.

**Orchestrated (you are a dispatched worker, e.g. issue-orchestration reviewer):** no human. Do not edit code, commit, or push. Put AMBIGUOUS items in the report and use the Orca `ask` command only if one blocks the verdict. In step 8 post only the summary as `gh pr comment <n> --body-file <file>` (no formal `gh pr review`, no inline comments, no `--add-reviewer`); the coordinator owns the verdict and merge.

## 7. Verdict and report

- `REQUEST_CHANGES` (orchestrated token: `CHANGES_REQUESTED`): High/Med security risk, broken or missing requirement, hard standards violation, critical bug.
- `APPROVE` (`APPROVED`): no actionable blockers; nits and judgement-call smells only.
- `COMMENT`: advisory, or pending author clarification.

Standalone: confirm the verdict with the user before submitting. Orchestrated: return exactly one of `APPROVED` / `CHANGES_REQUESTED` plus the summary below.

Summary format (also the PR review body):

```markdown
## Swarm Code Review Summary

| Lens | Status | Issues | Top Finding |
|---|---|---|---|
| Standards & Minimalism | PASS/WARN/FAIL | n | ... |
| Spec & Scope | PASS/WARN/FAIL | n | ... |
| Security & Tenancy | PASS/WARN/FAIL | n | ... |
| QA & Edge Cases | PASS/WARN/FAIL | n | ... |

### Triage
- Actionable: X
- Ambiguous: Y
- Nits: Z

### Actionable Items
- `path:L10-L20`: <issue and impact>
  Remediation: <fix or suggestion block>

### Nits
- `path:L30`: <suggestion>
```

## 8. Post to GitHub (PRs, standalone only)

```sh
gh pr review <n> --<comment|request-changes|approve> --body "<summary>"
# 422 on self-review: fall back to
gh pr comment <n> --body "<summary>"
# inline suggestion:
gh api repos/:owner/:repo/pulls/<n>/comments -f body="<suggestion block>" -f commit_id="<head_sha>" -f path="<file>" -F line=<line>
# request yourself as reviewer (not assignee):
gh pr edit <n> --add-reviewer @me   # fallback: gh api repos/:owner/:repo/pulls/<n>/requested_reviewers -f reviewers[]=@me
```
