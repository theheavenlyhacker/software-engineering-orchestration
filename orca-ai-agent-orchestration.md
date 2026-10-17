# Orca ADE Multi-Agent GitHub Issue Orchestration

> A practical architecture for using **Orca ADE** as the orchestration layer while using **Matt Pocock's `mattpocock/skills`** inside Codex, Claude, OpenCode, and other coding agents.

## Goal

Automatically process GitHub issues using specialized AI roles:

```text
GitHub Issue
    ↓
Issue Analysis / Triage
    ↓
Implementation Agent
    ↓
Code Review Agent
    ↓
Changes Requested? ── Yes ──→ Implementation Agent ──→ Review Again
    ↓ No / Approved
Create Pull Request
    ↓
CI / Tests
    ↓
Merge Conflict? ── Yes ──→ Conflict Resolver ──→ Review + Test Again
    ↓ No
Merge Pull Request
    ↓
Close Issue
    ↓
Next GitHub Issue
    ↓
Repeat until no eligible open issues remain
```

---

# 1. Architecture

Use Orca for orchestration and lifecycle management, not as the place where all engineering knowledge lives.

```text
                    ORCA ADE
                       │
              Coordinator / Controller
                       │
       ┌───────────────┼─────────────────┐
       │               │                 │
    GitHub          Orca Tasks       Orca Workers
       │               │                 │
       │               │         ┌───────┴────────┐
       │               │         │                │
       │               │      Codex            Claude /
       │               │    Implementer       Antigravity
       │               │         │              Reviewer
       │               │         │                │
       │               │         └──────┬─────────┘
       │               │                │
       │               │          Worktree / Branch
       │               │
       └───────────────┴───────────────→ GitHub PR
```

### Responsibility split

## Orca ADE

Orca owns:

- run lifecycle
- tasks
- dependencies / DAGs
- worker spawning
- worker completion events
- worktrees
- agent selection
- retry state
- decision gates
- task handoffs

## Coordinator

The coordinator owns:

- fetching GitHub issues
- selecting an eligible issue
- creating Orca tasks
- assigning a role to each worker
- deciding what happens after review
- enforcing retry limits
- creating / checking pull requests
- deciding when an issue is complete
- moving to the next issue

## Coding agents

The underlying agent owns engineering execution.

Examples:

- Codex → implementation
- Claude / Antigravity → independent code review
- Codex → conflict resolution
- Claude → issue analysis

## Matt Pocock skills

`mattpocock/skills` supplies reusable engineering practices to the underlying agents.

Useful skills for this architecture include:

| Skill | Usage |
|---|---|
| `setup-matt-pocock-skills` | Configure the repository once |
| `triage` | Classify / validate issues |
| `implement` | Implement work described by tickets/specs |
| `implement-spec` | Implement a larger task graph |
| `tdd` | Red-green-refactor implementation loop |
| `code-review` | Independent standards + specification review |
| `diagnosing-bugs` | Debug failed CI / difficult regressions |
| `codebase-design` | Keep implementation aligned with architecture |
| `pr` | Generate a strong pull request description |
| `research` | Investigate uncertain technical requirements |
| `domain-modeling` | Maintain project vocabulary / domain model |
| `retro` | Improve the agent environment after work |

---

# 2. Install Matt Pocock's Skills

Repository:

```text
https://github.com/mattpocock/skills
```

## Codex

Install the plugin:

```bash
codex plugin marketplace add mattpocock/skills
codex plugin add mattpocock-skills@mattpocock
```

## Claude Code

```bash
claude plugin install mattpocock-skills@claude-plugins-official
```

## Other agents / editable local skills

For agents such as OpenCode, Cursor, Devin, Windsurf, Amp, Pi, or another supported agent:

```bash
npx skills@latest add mattpocock/skills -a <agent>
```

Examples:

```bash
npx skills@latest add mattpocock/skills -a opencode
```

or choose interactively:

```bash
npx skills@latest add mattpocock/skills
```

When prompted, include:

```text
setup-matt-pocock-skills
```

Then update copied skills later with:

```bash
npx skills@latest update
```

> Pick one installation method per agent. Avoid installing both the managed plugin and copied skill files into the same agent because you may end up with duplicate skills.

---

# 3. Configure the Repository Once

Run this from the repository using an agent where the skills are installed:

```text
/setup-matt-pocock-skills
```

Configure:

- GitHub as the issue tracker
- your issue labels
- the location for generated documentation

The setup skill may create repository-specific agent configuration such as:

```text
docs/
└── agents/
    ├── issue-tracker.md
    ├── domain.md
    └── triage-labels.md
```

It can also add agent-skill instructions to an existing:

```text
AGENTS.md
```

or:

```text
CLAUDE.md
```

Commit these files because they are repository-specific context for future workers.

---

# 4. Recommended Repository Structure

```text
project/
│
├── .orca/
│   ├── roles/
│   │   ├── coordinator.md
│   │   ├── issue-analyzer.md
│   │   ├── implementer.md
│   │   ├── reviewer.md
│   │   ├── ci-fixer.md
│   │   ├── conflict-resolver.md
│   │   └── merger.md
│   │
│   ├── policies/
│   │   ├── retry-policy.md
│   │   ├── merge-policy.md
│   │   └── issue-selection-policy.md
│   │
│   └── orchestration/
│       └── github-issue-loop.md
│
├── docs/
│   └── agents/
│       ├── issue-tracker.md
│       ├── domain.md
│       └── triage-labels.md
│
├── AGENTS.md
├── GLOSSARY.md
└── src/
```

The `.orca/roles/*` files define **who the worker is**.

Matt Pocock's skills define **how the worker should perform engineering work**.

Example:

```text
ROLE
+
GITHUB ISSUE
+
REPOSITORY CONTEXT
+
MATT POCOCK SKILLS
+
CURRENT WORKTREE
=
ORCA WORKER TASK
```

---

# 5. Agent Roles

## 5.1 Coordinator

`.orca/roles/coordinator.md`

```markdown
# Role: Engineering Orchestrator

You coordinate GitHub issue implementation using Orca workers.

You do not implement application features yourself unless explicitly required.

## Responsibilities

1. Fetch eligible GitHub issues.
2. Select exactly one issue for the sequential MVP workflow.
3. Create a dedicated worktree / branch.
4. Dispatch the issue analyzer.
5. Dispatch the implementation agent.
6. Wait for completion.
7. Dispatch an independent reviewer.
8. Route CHANGES_REQUESTED back to the implementer.
9. Enforce retry limits.
10. Create a pull request only after approval.
11. Check CI.
12. Dispatch a CI fixer when necessary.
13. Detect merge conflicts.
14. Dispatch the conflict resolver when necessary.
15. Re-review conflict resolutions.
16. Merge only when all required gates pass.
17. Confirm the GitHub issue is closed.
18. Continue with the next eligible issue.

Never allow an implementation agent to approve its own work.
```

---

## 5.2 Issue Analyzer

Suggested agent:

```text
Claude / Antigravity
```

Recommended skills:

```text
triage
research
domain-modeling
codebase-design
```

`.orca/roles/issue-analyzer.md`

```markdown
# Role: Issue Analysis Engineer

Analyze exactly one GitHub issue before implementation.

Use the repository's installed engineering skills when relevant.

## Tasks

1. Read the full issue.
2. Inspect related code.
3. Read repository agent instructions.
4. Understand domain terminology.
5. Identify acceptance criteria.
6. Identify affected modules.
7. Identify likely test seams.
8. Identify security or migration risk.
9. Determine whether the issue is sufficiently specified.

## Output

Return one of:

READY_FOR_IMPLEMENTATION

or

BLOCKED

When ready, include:

- concise problem statement
- acceptance criteria
- relevant files/modules
- implementation constraints
- test strategy
- risk notes
```

---

# 6. Implementation Agent

Suggested agent:

```text
Codex
```

Primary Matt Pocock skills:

```text
implement
tdd
codebase-design
diagnosing-bugs
```

`.orca/roles/implementer.md`

```markdown
# Role: Senior Implementation Engineer

Implement exactly one GitHub issue.

Use the repository's installed Matt Pocock engineering skills when applicable.

Prefer:

- implement
- tdd
- codebase-design
- diagnosing-bugs

## Rules

1. Read the issue and analysis first.
2. Inspect AGENTS.md / repository instructions.
3. Follow existing architecture.
4. Work only inside the assigned worktree.
5. Implement one coherent vertical slice at a time.
6. Prefer red-green-refactor through the tdd skill when a test seam exists.
7. Do not modify unrelated code.
8. Add or update tests.
9. Run linting, type checking, tests, and build where available.
10. Never merge the pull request.
11. Never self-approve.

## Completion

Return:

IMPLEMENTATION_COMPLETE

with:

- files changed
- tests added/updated
- commands run
- known limitations
- anything the reviewer should inspect closely
```

---

# 7. Independent Code Reviewer

Suggested agent:

```text
Claude / Antigravity
```

Primary skill:

```text
code-review
```

Useful additional skills:

```text
codebase-design
domain-modeling
```

`.orca/roles/reviewer.md`

```markdown
# Role: Independent Senior Code Reviewer

You did NOT implement this change.

Review the diff against the originating GitHub issue and repository standards.

Use the installed `code-review` skill.

Review two separate axes:

1. Standards
   - architecture
   - maintainability
   - correctness
   - security
   - error handling
   - types
   - tests
   - unnecessary complexity

2. Specification
   - issue acceptance criteria
   - missing requirements
   - unintended behavior changes

## Verdict

Return exactly one primary verdict:

APPROVED

or

CHANGES_REQUESTED

For every requested change include:

- severity
- file
- affected area
- problem
- required correction

Do not edit the implementation yourself.
```

---

# 8. CI Fixer

Suggested agent:

```text
Codex
```

Skills:

```text
diagnosing-bugs
tdd
implement
```

`.orca/roles/ci-fixer.md`

```markdown
# Role: CI Failure Resolution Engineer

Fix only failures related to the current issue / pull request.

Use `diagnosing-bugs` for non-obvious failures.

## Process

1. Read CI logs.
2. Reproduce the failure locally if possible.
3. Minimize the failing condition.
4. Identify root cause.
5. Fix the root cause, not just the symptom.
6. Add a regression test when appropriate.
7. Run affected checks locally.

Return:

CI_FIX_COMPLETE

or

ESCALATE
```

---

# 9. Merge Conflict Resolver

Suggested agent:

```text
Codex
```

Useful skills:

```text
diagnosing-bugs
codebase-design
tdd
```

`.orca/roles/conflict-resolver.md`

```markdown
# Role: Git Merge Conflict Resolution Engineer

Resolve conflicts for the current pull request without changing its intended scope.

## Rules

1. Fetch the current target branch.
2. Identify all conflicts.
3. Understand both sides before editing.
4. Preserve intended behavior from both branches where appropriate.
5. Never blindly accept ours/theirs for all conflicts.
6. Keep changes scoped to conflict resolution.
7. Run tests, type checking, and build after resolving.
8. Never merge the PR yourself.

Return:

RESOLVED

or

ESCALATE

After RESOLVED, the diff MUST return to the independent reviewer.
```

---

# 10. Pull Request Agent / Merge Agent

Suggested agent:

```text
Codex
```

Primary skill:

```text
pr
```

`.orca/roles/merger.md`

```markdown
# Role: Pull Request and Merge Engineer

Create and merge a pull request only after all quality gates pass.

Use the installed `pr` skill when preparing the PR description.

## Required gates

- implementation complete
- independent review approved
- tests pass
- type checking passes when applicable
- build passes when applicable
- CI passes
- no unresolved merge conflicts
- no blocking reviewer findings

## Merge policy

Use squash merge unless the repository explicitly requires another strategy.

Never override branch protection.
Never force-push to protected branches.
Never merge when required checks are red.
```

---

# 11. GitHub CLI Setup

Install and authenticate GitHub CLI:

```bash
gh auth login
```

Confirm:

```bash
gh auth status
```

Fetch open issues:

```bash
gh issue list \
  --state open \
  --json number,title,body,labels,url
```

A safer production workflow should only process issues carrying a specific label, for example:

```text
agent-ready
```

Then:

```bash
gh issue list \
  --state open \
  --label agent-ready \
  --json number,title,body,labels,url
```

This prevents the orchestrator from trying to implement every open issue automatically.

---

# 12. Start an Orca Run

Create one orchestration run for the queue-processing session:

```bash
orca orchestration run-create \
  --objective "Resolve eligible agent-ready GitHub issues through implementation, review, CI, and merge" \
  --json
```

Treat the installed Orca version's orchestration documentation as authoritative if flags differ.

A useful command is:

```bash
orca skills get orchestration --full
```

---

# 13. Issue Processing Flow

For each eligible GitHub issue:

```text
SELECT ISSUE
    ↓
CREATE WORKTREE
    ↓
ANALYZE ISSUE
    ↓
IMPLEMENT
    ↓
REVIEW
    ↓
┌──────────────────────────────┐
│ CHANGES_REQUESTED?           │
└──────────────┬───────────────┘
               │ Yes
               ↓
         IMPLEMENT FIX
               ↓
            REVIEW
               │
               └──────── loop with limit

APPROVED
    ↓
CREATE PR
    ↓
CI
    ↓
┌──────────────────────────────┐
│ CI FAILED?                   │
└──────────────┬───────────────┘
               │ Yes
               ↓
             FIX CI
               ↓
             REVIEW
               ↓
               CI

CI PASS
    ↓
CHECK MERGEABILITY
    ↓
CONFLICT?
  │      │
 Yes     No
  │      │
  ↓      │
RESOLVE  │
  ↓      │
REVIEW   │
  ↓      │
TEST     │
  └──┬───┘
     ↓
    MERGE
     ↓
VERIFY ISSUE CLOSED
     ↓
NEXT ISSUE
```

---

# 14. Create the Worktree

Use one worktree / branch per issue.

Suggested naming:

```text
agent/issue-42-refresh-token
```

Conceptually:

```text
main
├── agent/issue-42-refresh-token
├── agent/issue-43-password-reset
└── agent/issue-44-email-verification
```

For the first MVP, process only one issue at a time even though Orca can support more complex DAGs.

---

# 15. Start the Issue Analyzer

Example pattern:

```bash
orca orchestration worker-start \
  --spec "Read .orca/roles/issue-analyzer.md. Analyze GitHub issue #42 and return READY_FOR_IMPLEMENTATION or BLOCKED." \
  --worktree new-child \
  --name issue-42-analysis \
  --agent claude \
  --setup run \
  --json
```

Wait for completion:

```bash
orca orchestration check \
  --wait \
  --types worker_done,escalation,question \
  --timeout-ms 900000 \
  --json
```

If the result is:

```text
BLOCKED
```

do not implement automatically.

Add a label such as:

```text
agent-blocked
```

and continue to another eligible issue.

---

# 16. Start the Implementation Agent

Example:

```bash
orca orchestration worker-start \
  --spec "Read .orca/roles/implementer.md. Implement GitHub issue #42 using the issue analysis. Use the installed Matt Pocock engineering skills, especially implement/tdd/codebase-design where applicable." \
  --worktree issue-42 \
  --agent codex \
  --json
```

The Codex environment should already have `mattpocock/skills` installed.

This is the key design:

```text
ORCA
  launches
     ↓
CODEX WORKER
  that already knows
     ↓
MATT POCOCK SKILLS
```

Orca does not need to reimplement `tdd`, `code-review`, `pr`, etc.

---

# 17. Review Loop

After implementation completion, dispatch an independent reviewer:

```bash
orca orchestration worker-start \
  --spec "Read .orca/roles/reviewer.md. Independently review GitHub issue #42 using the installed code-review skill. Return APPROVED or CHANGES_REQUESTED." \
  --worktree issue-42 \
  --agent claude \
  --json
```

If:

```text
CHANGES_REQUESTED
```

send the findings back to the implementation worker.

Conceptual loop:

```text
Codex Implementation
        ↓
Claude code-review
        ↓
CHANGES_REQUESTED
        ↓
Codex fixes findings
        ↓
Claude code-review
        ↓
APPROVED
```

Never use:

```text
Codex implements
    ↓
Codex reviews itself
    ↓
Codex approves itself
```

for the final quality gate.

---

# 18. Retry Limits

Never allow an unbounded agent loop.

Recommended defaults:

```text
MAX_ANALYSIS_ATTEMPTS       = 2
MAX_IMPLEMENTATION_ATTEMPTS = 3
MAX_REVIEW_CYCLES           = 3
MAX_CI_FIX_ATTEMPTS         = 2
MAX_CONFLICT_ATTEMPTS       = 2
```

If exceeded:

```text
ESCALATE_TO_HUMAN
```

Apply a label such as:

```text
agent-needs-human
```

and stop processing that issue.

---

# 19. Create the Pull Request

Once review returns APPROVED:

```bash
gh pr create \
  --title "fix: resolve #42" \
  --body "Closes #42"
```

Prefer having the PR worker use the `pr` skill to prepare a richer body containing:

- concise summary
- visual / textual explanation of the change
- before/after evidence
- test evidence
- merge risk / blast radius

---

# 20. Check CI

```bash
gh pr checks
```

If CI fails:

```text
CI Failure
    ↓
CI Fixer
    ↓
diagnosing-bugs skill
    ↓
Fix
    ↓
Tests
    ↓
Independent Review
    ↓
CI Again
```

Do not merge simply because the implementation reviewer approved an earlier commit.

CI fixes modify the diff, therefore the relevant changed code should be reviewed again.

---

# 21. Merge Conflict Flow

Before merge, update mergeability information and detect conflicts.

If conflicts exist:

```text
Conflict Resolver
    ↓
Resolve conflict
    ↓
Tests
    ↓
Independent Reviewer
    ↓
APPROVED
    ↓
CI
    ↓
Merge
```

Never use:

```text
Conflict Resolver
    ↓
Merge Immediately
```

because conflict resolution can silently alter behavior.

---

# 22. Merge

Recommended default:

```bash
gh pr merge --squash --delete-branch
```

Use repository branch protections and required status checks.

Do not let agents bypass protections.

If the PR body contains:

```text
Closes #42
```

GitHub can close the issue automatically after the merge.

Then verify:

```bash
gh issue view 42 --json state
```

Expected:

```text
CLOSED
```

---

# 23. Continue Until Queue Is Empty

Fetch the next eligible issue:

```bash
gh issue list \
  --state open \
  --label agent-ready \
  --json number,title,body,labels,url
```

If another issue exists:

```text
NEXT ISSUE
    ↓
ANALYZE
    ↓
IMPLEMENT
    ↓
REVIEW
    ↓
PR
    ↓
CI
    ↓
MERGE
```

Otherwise:

```text
NO_ELIGIBLE_OPEN_ISSUES
    ↓
END RUN
```

---

# 24. Suggested State Machine

```text
QUEUED
  ↓
ANALYZING
  ├── BLOCKED ──────────────→ HUMAN_REQUIRED
  ↓
IMPLEMENTING
  ├── FAILED ───────────────→ RETRY / HUMAN_REQUIRED
  ↓
REVIEWING
  ├── CHANGES_REQUESTED ────→ IMPLEMENTING
  ↓
APPROVED
  ↓
PR_CREATED
  ↓
CI_RUNNING
  ├── FAILED ───────────────→ CI_FIXING
  │                              ↓
  │                           REVIEWING
  ↓
CI_PASSED
  ↓
MERGEABILITY_CHECK
  ├── CONFLICT ─────────────→ RESOLVING_CONFLICT
  │                              ↓
  │                           REVIEWING
  ↓
READY_TO_MERGE
  ↓
MERGED
  ↓
CLOSED
```

---

# 25. Recommended GitHub Labels

```text
agent-ready
agent-in-progress
agent-review
agent-blocked
agent-needs-human
agent-completed
```

Example lifecycle:

```text
agent-ready
    ↓
agent-in-progress
    ↓
agent-review
    ↓
agent-completed
```

On failure:

```text
agent-blocked
```

or:

```text
agent-needs-human
```

---

# 26. Suggested Skills Per Role

| Orca Role | Agent | Matt Pocock skills |
|---|---|---|
| Coordinator | Claude / Codex | `triage`, `research` |
| Issue Analyzer | Claude | `triage`, `research`, `domain-modeling`, `codebase-design` |
| Implementer | Codex | `implement`, `tdd`, `codebase-design`, `diagnosing-bugs` |
| Reviewer | Claude / Antigravity | `code-review`, `codebase-design`, `domain-modeling` |
| CI Fixer | Codex | `diagnosing-bugs`, `tdd`, `implement` |
| Conflict Resolver | Codex | `diagnosing-bugs`, `codebase-design`, `tdd` |
| PR / Merge Agent | Codex | `pr` |
| Retrospective Agent | Claude | `retro` |

---

# 27. Minimal MVP

Do not start with every role.

Start here:

```text
GitHub agent-ready Issue
        ↓
Codex Implementer
  skills: implement + tdd
        ↓
Claude Reviewer
  skill: code-review
        ↓
┌───────────────────────┐
│ CHANGES_REQUESTED?    │
└──────────┬────────────┘
           │ Yes
           ↓
        Codex Fix
           ↓
      Claude Review
           │
           └──── loop max 3

APPROVED
    ↓
Create PR using pr skill
    ↓
Check CI
    ↓
Merge
    ↓
Next Issue
```

Once reliable, add:

1. Issue Analyzer
2. CI Fixer
3. Conflict Resolver
4. Security reviewer
5. Documentation reviewer
6. Parallel issue processing
7. Retrospective agent

---

# 28. Example Coordinator Pseudocode

```ts
while (true) {
  const issue = await getNextAgentReadyIssue();

  if (!issue) {
    break;
  }

  await markIssue(issue, "agent-in-progress");

  const analysis = await runIssueAnalyzer(issue);

  if (analysis.status === "BLOCKED") {
    await markIssue(issue, "agent-blocked");
    continue;
  }

  let approved = false;

  for (let attempt = 1; attempt <= 3; attempt++) {
    await runImplementer(issue, analysis);

    const review = await runReviewer(issue);

    if (review.verdict === "APPROVED") {
      approved = true;
      break;
    }

    await sendReviewFindingsToImplementer(review.findings);
  }

  if (!approved) {
    await markIssue(issue, "agent-needs-human");
    continue;
  }

  const pr = await createPullRequest(issue);

  let ciPassed = false;

  for (let attempt = 1; attempt <= 2; attempt++) {
    const ci = await checkCI(pr);

    if (ci.passed) {
      ciPassed = true;
      break;
    }

    await runCIFixer(issue, pr, ci);

    const postFixReview = await runReviewer(issue);

    if (postFixReview.verdict !== "APPROVED") {
      await sendReviewFindingsToImplementer(postFixReview.findings);
    }
  }

  if (!ciPassed) {
    await markIssue(issue, "agent-needs-human");
    continue;
  }

  if (await hasMergeConflict(pr)) {
    const resolved = await runConflictResolver(pr);

    if (!resolved) {
      await markIssue(issue, "agent-needs-human");
      continue;
    }

    const conflictReview = await runReviewer(issue);

    if (conflictReview.verdict !== "APPROVED") {
      await markIssue(issue, "agent-needs-human");
      continue;
    }

    if (!(await checkCI(pr)).passed) {
      await markIssue(issue, "agent-needs-human");
      continue;
    }
  }

  await mergePullRequest(pr);
  await verifyIssueClosed(issue);
  await markIssue(issue, "agent-completed");
}
```

---

# 29. Security and Safety Rules

Never give the orchestration system unrestricted permission without guardrails.

Recommended restrictions:

- only process issues labeled `agent-ready`
- never merge directly to protected branches
- require CI
- require independent review
- restrict GitHub token scopes
- never allow agents to change repository secrets
- never allow agents to disable branch protection
- never allow agents to delete unrelated branches
- never force push protected branches
- do not expose production credentials to implementation workers
- treat infrastructure changes as human approval gates
- use retry limits
- preserve logs for every Orca task

For high-risk repositories, require human approval before merge:

```text
AI APPROVED
   ↓
CI PASSED
   ↓
HUMAN APPROVAL
   ↓
MERGE
```

---

# 30. Recommended Final Architecture

```text
                         GitHub
                            │
                      agent-ready
                            │
                            ▼
                    ┌──────────────┐
                    │ ORCA         │
                    │ Coordinator  │
                    └──────┬───────┘
                           │
                    Issue Analyzer
                    Claude + skills
                           │
                           ▼
                    Codex Implementer
              implement + tdd + codebase-design
                           │
                           ▼
                  Independent Reviewer
                    code-review skill
                           │
                ┌──────────┴──────────┐
                │                     │
        CHANGES_REQUESTED          APPROVED
                │                     │
                ▼                     ▼
          Codex Implementer       PR Agent
                │                 pr skill
                └──── review ────────┤
                                      ▼
                                      CI
                               ┌──────┴──────┐
                               │             │
                              FAIL          PASS
                               │             │
                               ▼             ▼
                           CI Fixer      Conflict?
                      diagnosing-bugs     │     │
                               │         Yes    No
                               │          │      │
                               ▼          ▼      │
                            Review     Resolver  │
                                         │      │
                                         ▼      │
                                       Review   │
                                         │      │
                                         └──┬───┘
                                            ▼
                                           Merge
                                            │
                                            ▼
                                        Close Issue
                                            │
                                            ▼
                                         Next Issue
```

---

# 31. Principle to Keep

The most important architectural rule is:

```text
Orca determines WHO works and WHEN.

Matt Pocock's skills determine HOW engineering work is performed.
```

Do not duplicate the full engineering methodology inside your Orca coordinator.

Keep the coordinator small and deterministic:

```text
Select → Dispatch → Wait → Evaluate → Route → Merge → Repeat
```

Keep engineering knowledge composable inside agent skills:

```text
implement
tdd
code-review
diagnosing-bugs
codebase-design
pr
```

This keeps your orchestration understandable, replaceable, and model-independent.

---

# 32. Next Implementation Step

Implement the MVP in this order:

```text
1. Install mattpocock/skills in Codex and reviewer agent
2. Run setup-matt-pocock-skills
3. Configure GitHub `agent-ready` label
4. Create .orca/roles/implementer.md
5. Create .orca/roles/reviewer.md
6. Create one Orca Run
7. Fetch one GitHub issue with gh
8. Launch Codex implementation worker
9. Wait for worker_done
10. Launch independent review worker
11. Implement APPROVED / CHANGES_REQUESTED routing
12. Add max 3 review cycles
13. Create PR
14. Check CI
15. Merge
16. Verify issue closed
17. Fetch next issue
18. Loop
```

Only after this works reliably should you add parallel execution and additional specialized agents.
