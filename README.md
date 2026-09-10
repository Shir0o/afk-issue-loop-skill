# afk-issue-loop

An [Antigravity](https://antigravity.dev) skill that runs an autonomous, cost-minimal GitHub issue loop — triage unlabeled issues, implement `ready-for-agent` issues, open PRs, watch CI, fix until green, and squash-merge. One subagent at a time, no human babysitting required.

## What it does

- **Triage** — labels unlabeled issues (`needs-triage → needs-info / ready-for-agent / wontfix`), closes exact duplicates inline.
- **Implement** — branches off `main`, writes code TDD-style, opens a PR, watches CI, fixes failures (up to 3 cycles), then either merges or hands back to the orchestrator.
- **Orchestrate** — runs issues serially in dependency order; GitHub labels/comments/PRs are the only durable state (no local bookkeeping file).

## Requirements

- [Antigravity](https://antigravity.dev) (AGY)
- `gh` CLI authenticated with `repo` + `workflow` scopes
- A GitHub repo with the label vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix` (or a `docs/agents/triage-labels.md` that defines your own)

## Installation

Copy `SKILL.md` into your global Antigravity skills folder:

```bash
mkdir -p ~/.gemini/config/skills/afk-issue-loop
curl -fsSL https://raw.githubusercontent.com/Shir0o/afk-issue-loop-skill/main/SKILL.md \
  -o ~/.gemini/config/skills/afk-issue-loop/SKILL.md
```

Or clone and symlink:

```bash
git clone https://github.com/Shir0o/afk-issue-loop-skill.git
mkdir -p ~/.gemini/config/skills/afk-issue-loop
ln -s "$(pwd)/afk-issue-loop-skill/SKILL.md" ~/.gemini/config/skills/afk-issue-loop/SKILL.md
```

### Repo-scoped install (share with your team)

Commit it into `.agents/skills/` at your repo root — everyone who clones gets it automatically:

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

## Optional repo docs the skill respects

| File | Purpose |
|---|---|
| `docs/agents/issue-tracker.md` | Custom label vocabulary, PR-as-request-surface flag |
| `docs/agents/triage-labels.md` | Label definitions |
| `AGENTS.md` | Coding standards, full-gate commands (typecheck, lint, test, build) |
| `CONTEXT.md` | Domain model / codebase orientation |
| `docs/adr/` | Architecture decisions |

## License

MIT
