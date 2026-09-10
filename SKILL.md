---
name: afk-issue-loop
description: "Autonomous rinse-and-repeat loop: process GitHub issues end-to-end (triage unlabeled issues, implement ready-for-agent issues, PR, watch CI, fix until green, squash-merge, sync main). One subagent at a time, cost-minimal. Use when the user asks to run the issue loop, process open issues AFK, or rinse-and-repeat the issue/PR cycle."
disable-model-invocation: true
argument-hint: "[issue numbers or filter, default: all actionable open issues]"
---

# AFK Issue Loop

Autonomous loop over GitHub issues for the current repo. One issue per subagent;
strictly serial; GitHub is the only durable state.

## Operating principles (hard rules)

1. **Automation, not concurrency.** At most ONE working subagent at any moment.
   Spawn the next only after the previous settles and its PR is merged (or blocked).
2. **Cost-minimal bookkeeping.** The orchestrator does cheap `gh` operations inline
   (closing duplicates, labels, comments, epic closure). Subagents are only for
   issue-sized work (triage analysis, implementation). Never spawn a subagent for a
   one-command change.
3. **Dependency order.** Within a family of related issues (spec + decomposition
   tickets, ADR chains), implement in dependency order: prerequisites first. Close
   parent/epic issues only after their chain merges. Independent issues last.
4. **Never push to `main` directly.** Everything lands via PR + checks.
5. **Honesty over completion.** Blocked (flaky CI >3 attempts, missing credentials,
   genuinely ambiguous spec) → stop that issue, report `blocked`, move on. Never
   fake a merge or a triage outcome.
6. **Ephemeral agents.** Each subagent gets a complete, self-contained prompt (see
   template below) and returns a short report. Its transcript is discarded — all
   durable state lives on GitHub (labels, comments, PRs).

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
- **Queue**: `gh issue list --state open` → order:
  1. unlabeled (triage) and `needs-triage` issues,
  2. `ready-for-agent` issues (dependency order within families),
  3. everything else (`needs-info`, `ready-for-human` — skip; report only).
- **Duplicates**: same title/body/author within seconds apart → close the emptier
  one as duplicate of the fuller one, inline, label `wontfix`, comment links both.

## Phase 1 — Per-issue dispatch

For each queued issue, in order, spawn ONE subagent with the template below.
While it runs, the orchestrator does nothing else agent-wise; you may answer user
questions and do inline `gh` bookkeeping.

Branch on labels:
- **Has `ready-for-agent`** → Branch A (implement).
- **Unlabeled / `needs-triage`** → Branch B (triage only).
- **`needs-info`, `ready-for-human`, `wontfix`** → skip.

### Subagent prompt template

Fill `<...>` from the repo profile. Give the subagent the repo docs list, the
required-check names, and the merge method — it has no conversation history.

```
# Goal
Process GitHub issue <N> of <OWNER>/<REPO> through the AFK issue loop.

# Constraints
- Shared git worktree at <PATH>. You <OWN | DO NOT TOUCH> git HEAD
  (OWN for implement; DO NOT TOUCH for triage).
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
3. If the agent died mid-flight (`failed`), inspect the worktree before
   re-dispatching: uncommitted work is recovered by the next agent prompt (list
   exactly what exists and what remains); a clean tree means fresh dispatch. Never
   re-dispatch blind onto a dirty tree.
4. If the user's primary checkout (`~/<repo>`) has staged/uncommitted changes, do
   not commit or discard them — `git stash push -m "<descriptive note>"` before any
   sync, and tell the user what was stashed and why.
5. Mark the issue done; if it unblocked dependents, reorder the queue.
6. Spawn the next subagent. Repeat until the queue is empty.

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