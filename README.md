# afk-issue-loop

> An autonomous, cost-minimal GitHub issue loop for AI coding agents — triage unlabeled issues, implement `ready-for-agent` issues, open PRs, watch CI, fix until green, and squash-merge. Run serially or spawn parallel agents. No human babysitting required.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## What it does

- **Triage** — labels unlabeled issues (`needs-triage → needs-info / ready-for-agent / wontfix`), closes exact duplicates inline.
- **Implement** — branches off `main`, writes code TDD-style, opens a PR, watches CI, fixes failures (up to 3 cycles), then merges or hands off to the orchestrator.
- **Orchestrate** — runs serially (1 subagent at a time) or in parallel ($N$ concurrent workers for independent issues, partitioned to prevent merge conflicts).
- **Interruption Resilience** — resumes in-flight subagents by inspecting session history and worktree diffs rather than starting from scratch after network drops or process stops.

## How it works

`SKILL.md` is a natural-language workflow file. Point your AI coding agent at it and it will follow the three-phase loop:

```
Orchestrator (the agent)
  │
  ├─ Phase 0: ground repo profile, scan in-flight work, & ask execution mode
  │    gh repo view, check existing worktrees/branches/PRs, open issue queue
  │    prompt user: serial (1 agent) or parallel (N concurrent agents)
  │
  ├─ Phase 1: issue dispatch (serial or conflict-minimized parallel across worktrees)
  │    ├─ interrupted in-flight   →  resume with worktree diff context
  │    ├─ unlabeled / needs-triage →  triage workflow (Branch B)
  │    └─ ready-for-agent          →  implement + tdd + code-review (Branch A)
  │
  ├─ Phase 2: after each task settles
  │    verify gh claim → merge (squash) → sync main → dispatch next issue(s)
  │
  └─ Phase 3: wrap-up
       close epics, report table of outcomes
```

## Dependencies

This workflow calls into four companion skill files from [Matt Pocock's engineering skill suite](https://github.com/mattpocock/skills). **All four must be available to your agent before using this workflow.**

| Skill | What it does in this loop |
|---|---|
| [`triage`](https://github.com/mattpocock/skills/tree/main/triage) | Labels unlabeled/needs-triage issues, closes duplicates |
| [`implement`](https://github.com/mattpocock/skills/tree/main/implement) | Branches, writes code, opens PR |
| [`tdd`](https://github.com/mattpocock/skills/tree/main/tdd) | Red-green-refactor cycle used by the implement step |
| [`code-review`](https://github.com/mattpocock/skills/tree/main/code-review) | Self-review before pushing PR |

## Requirements

- An AI coding agent capable of reading and following markdown workflow files
- `gh` CLI authenticated with `repo` + `workflow` scopes
- A GitHub repo with the label vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`
  (or a `docs/agents/triage-labels.md` defining your own)

## Installation

### 1. Install the dependency skills

Download the four companion skills into wherever your agent loads workflow files from. Example using a `.agents/skills/` folder at the repo root:

```bash
for skill in triage implement tdd code-review; do
  mkdir -p .agents/skills/$skill
  curl -fsSL "https://raw.githubusercontent.com/mattpocock/skills/main/$skill/SKILL.md" \
    -o .agents/skills/$skill/SKILL.md
done
```

Or follow the install instructions in [Matt Pocock's skills repo](https://github.com/mattpocock/skills).

### 2. Install this workflow

```bash
mkdir -p .agents/skills/afk-issue-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-issue-loop-skill/main/SKILL.md \
  -o .agents/skills/afk-issue-loop/SKILL.md
```

Commit and push so your whole team gets it:

```bash
git add .agents/
git commit -m "chore: add afk-issue-loop skill and dependencies"
```

> **Note:** The exact install path depends on your agent. `.agents/skills/` is a common convention. Check your agent's docs for where it loads workflow files from.

## Usage

Tell your agent to run the loop. Exact phrasing depends on your agent, but any of these work:

```
run the issue loop
process open issues AFK
```

The agent reads `SKILL.md`, grounds the repo profile (merge policy, required CI checks, label config), asks if you prefer sequential execution (1 agent, cost-minimal; default) or parallel execution ($N$ agents across isolated worktrees), and then works through the queue until it's empty.

You can also specify parallel execution directly:

```
run the issue loop in parallel
run the issue loop with 3 parallel agents
```

Target specific issues:

```
run the issue loop for #12 #15 #20
```

## Optional repo docs the workflow respects

| File | Purpose |
|---|---|
| `docs/agents/issue-tracker.md` | Custom label vocabulary, PR-as-request-surface flag |
| `docs/agents/triage-labels.md` | Label definitions |
| `AGENTS.md` | Coding standards, full-gate commands (`typecheck && lint && test && build`) |
| `CONTEXT.md` | Domain model / codebase orientation |
| `docs/adr/` | Architecture decisions |

These files are also read by the dependency skills. The `setup-matt-pocock-skills` skill can scaffold them for you.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
