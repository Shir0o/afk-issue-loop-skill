# Changelog

All notable changes to this skill will be documented here.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)

## [Unreleased]

### Added
- Integration with Matt Pocock's `/pr` skill (`skill://pr`) for writing PR bodies with summary diagrams/diffs, before/after evidence, and merge danger analysis
- Interruption recovery & resumption: Phase 0 scans in-flight worktrees, branches, PRs, and session history
- Targeted Resume dispatch prompt providing worktree path, uncommitted diffs, commits, and remaining tasks to avoid restarting from scratch
- Conflict-minimizing parallel issue selection: partitions candidate issues by disjoint subsystems/files and strictly sequences dependencies to prevent merge conflicts and rebase cycles
- Triage bypass for `to-spec` and `to-tickets`: auto-detects spec and ticket structural signatures in issue bodies, skips Branch B (triage), labels `ready-for-agent` inline, and routes directly to Branch A (implement)
- User prompt in Phase 0 to choose sequential execution (1 agent, cost-minimal; default) or parallel execution ($N$ agents across isolated worktrees)
- Support for CLI arguments to set execution mode or concurrency limit (`--sequential`, `--serial`, `--parallel`, `--concurrency <N>`)
- Parallel dispatch logic for independent issues with isolated worktrees and sequential orchestrator merges
- Scoped worktree cleanup (Principle 8): actively registers worktrees spawned during the session, automatically cleans up settled worktrees (`.worktrees/issue-<N>`) and branches (`agent/issue-<N>`) in Phase 2 and sweeps in Phase 3, while preserving worktrees for blocked/interrupted issues and never touching worktrees created by other agents or users

### Added
- Initial release of `afk-issue-loop` skill
- Phase 0: repo profile grounding (merge policy, required checks, label config, issue queue)
- Phase 1: per-issue dispatch — triage branch (B) and implement branch (A)
- Phase 2: post-agent verification, squash-merge, main sync
- Phase 3: epic closure and outcome report table
- Orchestrator-merges mode (default) vs agent-merges mode
- Dependency on `triage`, `implement`, `tdd`, `code-review` skills
