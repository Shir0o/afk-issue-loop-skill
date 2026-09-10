# afk-issue-loop

> An [Antigravity](https://antigravity.dev) skill that runs an autonomous, cost-minimal GitHub issue loop — triage unlabeled issues, implement `ready-for-agent` issues, open PRs, watch CI, fix until green, and squash-merge. One subagent at a time. No human babysitting required.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## What it does

- **Triage** — labels unlabeled issues (`needs-triage → needs-info / ready-for-agent / wontfix`), closes exact duplicates inline.
- **Implement** — branches off `main`, writes code TDD-style, opens a PR, watches CI, fixes failures (up to 3 cycles), then merges or hands off to the orchestrator.
- **Orchestrate** — runs issues serially in dependency order. GitHub labels, comments, and PRs are the only durable state — no local bookkeeping file.

## Dependencies

This skill orchestrates four companion skills from [Matt Pocock's engineering skill suite](https://github.com/mattpocock/skills). **All four must be installed before using this skill.**

| Skill | What it does in this loop |
|---|---|
| [`triage`](https://github.com/mattpocock/skills/tree/main/triage) | Labels unlabeled/needs-triage issues, closes duplicates |
| [`implement`](https://github.com/mattpocock/skills/tree/main/implement) | Branches, writes code, opens PR |
| [`tdd`](https://github.com/mattpocock/skills/tree/main/tdd) | Red-green-refactor cycle used by the implement subagent |
| [`code-review`](https://github.com/mattpocock/skills/tree/main/code-review) | Self-review before pushing PR |

Install them all at once via the `setup-matt-pocock-skills` skill, or individually — see [Installation](#installation).

## Requirements

- [Antigravity](https://antigravity.dev) (AGY)
- `gh` CLI authenticated with `repo` + `workflow` scopes
- A GitHub repo with the label vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`
  (or a `docs/agents/triage-labels.md` defining your own)

## Installation

### 1. Install the dependencies (Matt Pocock's skills)

If you already have Matt's skills installed, skip to step 2.

```bash
# Install all four dependency skills at once
for skill in triage implement tdd code-review; do
  mkdir -p ~/.gemini/config/skills/$skill
  curl -fsSL "https://raw.githubusercontent.com/mattpocock/skills/main/$skill/SKILL.md" \
    -o ~/.gemini/config/skills/$skill/SKILL.md
done
```

Or run the `setup-matt-pocock-skills` skill inside an Antigravity conversation:

```
set up Matt Pocock's skills
```

### 2. Install this skill

```bash
mkdir -p ~/.gemini/config/skills/afk-issue-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-issue-loop-skill/main/SKILL.md \
  -o ~/.gemini/config/skills/afk-issue-loop/SKILL.md
```

### Repo-scoped install (share with your team)

Commit the skill into `.agents/skills/` at your repo root — everyone who clones it gets it automatically:

```bash
mkdir -p .agents/skills/afk-issue-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-issue-loop-skill/main/SKILL.md \
  -o .agents/skills/afk-issue-loop/SKILL.md
git add .agents/skills/afk-issue-loop/SKILL.md
git commit -m "chore: add afk-issue-loop skill"
```

## Usage

Open an Antigravity conversation in your repo and say:

```
run the issue loop
```

or

```
process open issues AFK
```

The agent reads `SKILL.md`, grounds the repo profile (merge policy, required CI checks, label config), then works through the queue serially until it's empty.

You can also target specific issues:

```
run the issue loop for #12 #15 #20
```

## Optional repo docs the skill respects

| File | Purpose |
|---|---|
| `docs/agents/issue-tracker.md` | Custom label vocabulary, PR-as-request-surface flag |
| `docs/agents/triage-labels.md` | Label definitions |
| `AGENTS.md` | Coding standards, full-gate commands (`typecheck && lint && test && build`) |
| `CONTEXT.md` | Domain model / codebase orientation |
| `docs/adr/` | Architecture decisions |

These files are read by both this skill and the dependency skills. The `setup-matt-pocock-skills` skill scaffolds them for you.

## How it works

```
Orchestrator (you)
  │
  ├─ Phase 0: ground repo profile
  │    gh repo view, .github/workflows, merge policy, open issue queue
  │
  ├─ Phase 1: per-issue dispatch (serial, one subagent at a time)
  │    ├─ unlabeled / needs-triage  →  subagent: skill://triage (Branch B)
  │    └─ ready-for-agent          →  subagent: skill://implement + tdd + code-review (Branch A)
  │
  ├─ Phase 2: after each agent settles
  │    verify gh claim → merge (squash) → sync main → next issue
  │
  └─ Phase 3: wrap-up
       close epics, report table of outcomes
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
