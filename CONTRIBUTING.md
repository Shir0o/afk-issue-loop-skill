# Contributing to afk-issue-loop

Thanks for your interest in contributing! This is a natural-language workflow file (`SKILL.md`) for AI coding agents. The entire skill is a single markdown file — no build step, no dependencies to install for development.

## How to contribute

### Reporting bugs or suggesting improvements

Open a [GitHub Issue](https://github.com/Shir0o/afk-issue-loop-skill/issues). Please include:

- What you asked the agent to do
- What it actually did (paste the agent's output or a transcript excerpt)
- What you expected instead
- Which AI coding agent you used
- The dependency skills installed (`triage`, `implement`, `tdd`, `code-review`)

### Proposing changes to `SKILL.md`

1. Fork the repo
2. Create a branch: `git checkout -b fix/describe-your-change`
3. Edit `SKILL.md`
4. Test it: load the skill into your agent, point it at a real repo, and run `process open issues AFK`
5. Open a PR — describe what you changed, why, and what you tested

### What makes a good change

- **Tighter constraints** — the hard rules (isolated worktrees per worker, no direct push to `main`, sequential main merges, ≤3 fix cycles) exist to keep outcomes predictable and avoid conflicts. Changes that weaken them need a very strong justification.
- **Better prompts** — the subagent prompt template is the core of the skill. Improvements that make agent outputs more reliable or predictable are very welcome.
- **Clearer phase boundaries** — the three-phase structure (ground profile → dispatch → wrap-up) should stay legible and easy for future maintainers to reason about.

### What's out of scope

- Uncoordinated or conflicting concurrent writes to the same branch/worktree
- Support for non-GitHub issue trackers (use Matt Pocock's `triage` skill config for that)
- Features unrelated to the issue → PR → merge lifecycle

## Development setup

No build step. The entire workflow is `SKILL.md`.

To test your changes, copy the file into wherever your agent loads workflow/skill files from:

```bash
cp SKILL.md /path/to/your/agent/skills/afk-issue-loop/SKILL.md
```

Then run it in a repo that has the dependency skills installed.

## Dependency skills

This workflow calls into four skills from Matt Pocock's suite. If you find a bug that originates in one of those skills, please open an issue in the appropriate repo rather than here.

| Skill | Repo |
|---|---|
| `triage` | https://github.com/mattpocock/skills |
| `implement` | https://github.com/mattpocock/skills |
| `tdd` | https://github.com/mattpocock/skills |
| `code-review` | https://github.com/mattpocock/skills |

## Code of conduct

Be kind. This is a small open-source project; everyone here is a volunteer.
