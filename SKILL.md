---
name: afk-issue-loop
description: "Autonomous rinse-and-repeat loop: process GitHub issues end-to-end (triage unlabeled issues, implement ready-for-agent issues, PR, watch CI, fix until green, squash-merge, sync main). Configurable serial or parallel agents, cost-minimal."
argument: "[issue numbers or filter, e.g. --parallel or --concurrency N; default: all actionable open issues]"
---

# AFK Issue Loop

Autonomous loop over GitHub issues for the current repo. One issue per subagent;
serial by default or parallel upon user confirmation; GitHub is the only durable state.

## Operating principles (hard rules)

1. **User-selected concurrency (default sequential).** Before dispatching, ask
   the user whether to run sequentially (1 agent at a time, cost-minimal; default)
   or in parallel (specifying maximum concurrency $N$, e.g. 2–3 agents). If arguments
   specify `--parallel`, `--concurrency <N>`, or `--sequential`/`--serial`, honor
   that without re-asking.
   - In sequential/serial mode: at most ONE working subagent at any moment. Spawn the next only
     after the previous settles.
   - In parallel mode: spawn up to $N$ subagents concurrently for independent issues.
     Each subagent MUST operate in its own isolated git worktree/branch.
2. **Cost-minimal bookkeeping.** The orchestrator does cheap `gh` operations inline
   (closing duplicates, labels, comments, epic closure). Subagents are only for
   issue-sized work (triage analysis, implementation). Never spawn a subagent for a
   one-command change.
3. **Conflict-minimizing sequence & dependency order.**
   - Within a family of related issues (spec + decomposition tickets, ADR chains),
     strictly implement in dependency order: prerequisites first. Close parent/epic
     issues only after their chain merges.
   - When selecting issues for parallel dispatch, optimize to prevent merge conflicts
     and avoid costly rebases: partition concurrent issues by disjoint files, modules,
     or directory subsystems (e.g. backend vs frontend, separate packages, independent
     endpoints). Never dispatch issues concurrently if they touch or are expected to
     modify overlapping files or shared schemas.
4. **Never push to `main` directly.** Everything lands via PR + checks. Merges to
   `main` are performed sequentially by the orchestrator.
5. **Honesty over completion.** Blocked (flaky CI >3 attempts, missing credentials,
   genuinely ambiguous spec) → stop that issue, report `blocked`, move on. Never
   fake a merge or a triage outcome.
6. **Resumption over restart (interruption resilience).** While subagents are
   ephemeral during uninterrupted runs, if the orchestrator or subagent is interrupted
   and resumed later (due to connection drops, process restarts, or user termination),
   the orchestrator MUST NOT start anew or discard ongoing work. Instead:
   - Inspect chat/session history, existing worktrees (`.worktrees/issue-*`), active branches
     (`agent/issue-*`), and open PRs.
   - Dispatch subagents with a resume prompt referencing the existing worktree, unstaged/committed
     changes, and remaining goals.
   - Rebuilding from scratch is a last-resort fallback only if an in-flight worktree is
     unrecoverably corrupt.
7. **Ephemeral baseline.** Completed subagent transcripts are discarded once settled —
   all durable state lives on GitHub (labels, comments, PRs).

## Phase 0 — Ground the repo profile (orchestrator, inline, once per run)

Gather and hold these facts; they parameterize every subagent prompt:

- **Tracker config**: read `docs/agents/issue-tracker.md` and
  `docs/agents/triage-labels.md` if present (label vocabulary, whether PRs are a
  request surface). Fall back to the canonical labels: `needs-triage`, `needs-info`,
  `ready-for-agent`, `ready-for-human`, `wontfix`.
- **Issue-tracker doc paths**: `AGENTS.md`, `CONTEXT.md`, `docs/adr/`, `docs/agents/*.md`,
  `.out-of-scope/` — instruct subagents to respect whichever exist.
- **Merge method**: `gh repo view --json squashMergeAllowed,...` — prefer squash
  if allowed; else match the repo's most recent merged PRs.
- **Required checks**: from `.github/workflows/*` (job names) or
  `gh pr checks` on the last merged PR. The merge gate is ALL required checks green.
- **Merge policy**: `gh api repos/<OWNER>/<REPO>/rules/branches/main` (fall back to
  `gh api .../branches/main/protection`) — required approvals, merge queue, admin
  bypass. If a review requirement exists and no human review is expected during the
  run, decide the merge authority UP FRONT (below) rather than letting the first
  green PR discover it.
- **Install state**: if `node_modules` is missing, run the install (`npm ci` or
  equivalent) in the background BEFORE the first implement agent — don't make
  agents race on it.
- **In-flight / Interruption scan**: Check whether the loop was interrupted previously
  (process kill, connection drop, timeout):
  1. Inspect existing worktrees (`.worktrees/issue-*` or `git worktree list`).
  2. Inspect local and remote branches (`git branch -a | grep 'agent/issue-'`).
  3. Inspect open PRs created by the loop (`gh pr list --head 'agent/issue-'`).
  4. Inspect recent chat / agent session history or subagent logs to determine the last
     active issue and progress.
  Any issue found in-flight is marked for **Resumption** in Phase 1 rather than fresh dispatch.
- **Queue**: `gh issue list --state open` → order:
  1. In-flight / interrupted issues (resume existing work first),
  2. unlabeled (triage) and `needs-triage` issues,
  3. `ready-for-agent` issues (dependency order within families, then conflict-minimized grouping),
  4. everything else (`needs-info`, `ready-for-human` — skip; report only).
- **Duplicates**: same title/body/author within seconds apart → close the emptier
  one as duplicate of the fuller one, inline, label `wontfix`, comment links both.
- **Execution mode (Ask user, default sequential)**: If arguments do not specify
  (`--parallel`, `--serial`, `--sequential`, or `--concurrency <N>`), ask the user:
  > "Do you want to run issues sequentially (1 agent at a time, cost-minimal; default) or in parallel (specify concurrency limit, e.g. 2 or 3)?"
  If the user does not specify a preference or chooses default, use `sequential`. If parallel is chosen without a number, default concurrency limit $N$ to 2.
  Record the selected mode (`sequential` or `parallel`) and concurrency limit $N$.

## Phase 1 — Issue dispatch (sequential or parallel)

Depending on the chosen execution mode:
- **Sequential mode**: For each queued issue in order, spawn ONE subagent with the appropriate
  template below (Fresh or Resume). While it runs, do inline `gh` bookkeeping only. Spawn the next only after the current settles.
- **Parallel mode (Conflict-Minimizing Issue Selection)**:
  Identify independent issues in the queue (no unmet dependencies). To minimize cost,
  prevent git merge conflicts, and avoid wasted CI/fix cycles:
  1. **Disjoint Subsystems**: Group candidates by domain / directory / module (e.g. backend vs
     frontend, distinct API routes, separate config/doc files). Select issues that touch
     mutually exclusive file sets.
  2. **Never Concurrently Dispatch Overlapping Changes**: If two issues require changes to
     the same files, shared models, or schema definitions, execute them in sequence, not in
     parallel.
  3. **Strict Sequencing**: For dependent or related issues, merge prerequisites first so subsequent
     agents branch off updated code cleanly.
  Spawn up to $N$ subagents concurrently across isolated git worktrees (e.g., `.worktrees/issue-<N>`).

Branch on labels:
- **Has `ready-for-agent`** → Branch A (implement).
- **Unlabeled / `needs-triage`** → Branch B (triage only).
- **`needs-info`, `ready-for-human`, `wontfix`** → skip.

### Subagent prompt templates

Fill `<...>` from the repo profile. Give the subagent the repo docs list, the
required-check names, and the merge method — it has no conversation history.

#### 1. Fresh dispatch prompt

```
# Goal
Process GitHub issue <N> of <OWNER>/<REPO> through the AFK issue loop.

# Constraints
- Git worktree / branch at <PATH>. You <OWN | DO NOT TOUCH> git HEAD
  (OWN for implement; DO NOT TOUCH for triage). In parallel mode, work only in your isolated worktree.
- Follow the standing SOP: execute skill://afk-issue-loop, Branch <A|B>, for
  issue <N> only. Skills: skill://triage (+ skill://triage/AGENT-BRIEF.md),
  skill://implement, skill://tdd, skill://code-review — read what your branch needs.
- Respect repo docs: <AGENTS.md, CONTEXT.md, docs/adr/, docs/agents/*.md — list what exists>.
- Required checks: <NAMES>. Merge method: <squash|merge|rebase>.
- Full gate before push: <COMMANDS from AGENTS.md/CI, e.g. typecheck && lint && test:coverage && build>.
- Branch A: branch `agent/issue-<N>` from fresh origin/main; TDD per AGENTS.md;
  PR title in the repo's conventional-commit style; body contains `Closes #<N>`;
  `gh pr checks <pr> --watch`; fix on the same branch until green (≤3 fix cycles,
  then report blocked). Stop at green — do NOT merge unless the loop is in
  agent-merges mode. In orchestrator-merges mode (default when `main` has any
  review/merge-queue requirement, or when an agent in a prior run wasted budget on
  merge-policy errors): report `pushed (PR #x, all required checks green)` and stop;
  the orchestrator squash-merges (with `--admin` if needed) and syncs.
  After merge, main sync: if a local checkout has `main` checked out
  (multi-worktree setups), `git fetch origin main:main` fails — the orchestrator
  runs `git pull --ff-only` in that checkout instead.
- Branch B: triage per skill://triage with the AFK substitution (act autonomously
  when confident; `needs-info` + specific questions when genuinely ambiguous; never
  grill the reporter). Every comment starts with:
  `> *This was generated by AI during triage.*`
- Final message, exactly:
  ISSUE <N>: <merged (PR #x) | pushed (PR #x, checks green) | triaged (label) | closed (wontfix) | blocked>
  Notes: <1–3 lines>
```

#### 2. Resume dispatch prompt (when resuming interrupted work)

```
# Goal
RESUME GitHub issue <N> of <OWNER>/<REPO> through the AFK issue loop.
An earlier subagent was interrupted. Do NOT wipe the branch or restart from scratch.

# Worktree & Context State
- Existing worktree: <PATH> on branch `agent/issue-<N>`.
- Prior progress status:
  - Local commits: <SUMMARY OF COMMITS ON BRANCH>
  - Uncommitted changes / files: <GIT STATUS / DIFF SUMMARY>
  - PR status: <PR #x OPEN / CHECKS STATUS / NONE>
  - Interruption point from session history: <LAST KNOWN ACTION / MILESTONE>

# Constraints
- Work directly inside existing worktree <PATH>. Continue from the current state of files.
- Inspect the current code and test suite before making changes.
- If existing work is valid, build upon it: complete remaining implementation/tests, ensure the full gate passes, and push.
- If a PR already exists, push fixes to the existing branch; do not open a duplicate PR.
- If the worktree is in an unrecoverable state (corrupt rebase or syntax deadlock), run `git reset --hard` to the last clean commit or `origin/main` as a one-time fallback, and state that in the final report.
- Follow the standing SOP: execute skill://afk-issue-loop, Branch <A|B>, for issue <N> only.
- Full gate before push: <COMMANDS from AGENTS.md/CI, e.g. typecheck && lint && test:coverage && build>.
- Final message, exactly:
  ISSUE <N>: <merged (PR #x) | pushed (PR #x, checks green) | triaged (label) | closed (wontfix) | blocked>
  Notes: <1–3 lines>
```

### Orchestrator-side prompts during a run

- User answers to triage questions are relayed to a running agent via hub DM; once
  an agent settles, answers go on the GitHub issue itself — the next pass reads them.

## Phase 2 — After each agent settles

1. Verify the claim with one `gh` call — PR state (`gh pr view <pr> --json state`),
   issue state, labels, comment posted. NEVER trust `merged` in a report without it;
   an agent under budget pressure has reported "merged" for an unmerged PR.
2. In orchestrator-merges mode: merge (`gh pr merge <pr> --squash --admin` when the
   ruleset demands an approval), verify the issue auto-closed, and sync main —
   `git pull --ff-only` in the checkout that has `main` (a worktree cannot fetch
   into a branch checked out elsewhere).
3. If the agent died mid-flight (`failed` or process interrupted):
   - Inspect the worktree (`git status`, `git diff`, `git log -n 3`) and session logs.
   - If uncommitted or partially finished work exists, do not discard it. Dispatch a
     subagent using the **Resume dispatch prompt** with the recovered diff context.
   - Only if the tree is completely empty or cleanly aborted should a fresh dispatch be used.
4. If the user's primary checkout (`~/<repo>`) has staged/uncommitted changes, do
   not commit or discard them — `git stash push -m "<descriptive note>"` before any
   sync, and tell the user what was stashed and why.
5. Mark the issue done; if it unblocked dependents, reorder the queue.
6. Dispatch the next subagent(s):
   - In sequential mode: spawn the next single subagent.
   - In parallel mode: maintain up to $N$ active agents by dispatching the next available independent issue (following conflict-minimizing partitioning).
   Repeat until the queue is empty.

## Phase 3 — Wrap-up

- Epics/spec parents whose decomposition tickets all merged → close inline with a
  chain summary comment (AI disclaimer if triage-flavored).
- Report to the user: table of issue → outcome (merged PR #, triaged label, closed,
  blocked + reason), plus any human decisions waiting on GitHub (`needs-info`).

## Report format (subagents, mandatory)

```
ISSUE <N>: <merged (PR #x) | pushed (PR #x, checks green) | triaged (label) | closed (wontfix) | blocked>
Notes: <1–3 lines: what was done / what a human must decide>
```