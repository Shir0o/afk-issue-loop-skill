# Changelog

All notable changes to this skill will be documented here.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)

## [Unreleased]

### Added
- Interruption recovery & resumption: Phase 0 scans in-flight worktrees, branches, PRs, and session history
- Targeted Resume dispatch prompt providing worktree path, uncommitted diffs, commits, and remaining tasks to avoid restarting from scratch
- Conflict-minimizing parallel issue selection: partitions candidate issues by disjoint subsystems/files and strictly sequences dependencies to prevent merge conflicts and rebase cycles
- User prompt in Phase 0 to choose sequential execution (1 agent, cost-minimal; default) or parallel execution ($N$ agents across isolated worktrees)
- Support for CLI arguments to set execution mode or concurrency limit (`--sequential`, `--serial`, `--parallel`, `--concurrency <N>`)
- Parallel dispatch logic for independent issues with isolated worktrees and sequential orchestrator merges

### Added
- Initial release of `afk-issue-loop` skill
- Phase 0: repo profile grounding (merge policy, required checks, label config, issue queue)
- Phase 1: per-issue dispatch — triage branch (B) and implement branch (A)
- Phase 2: post-agent verification, squash-merge, main sync
- Phase 3: epic closure and outcome report table
- Orchestrator-merges mode (default) vs agent-merges mode
- Dependency on `triage`, `implement`, `tdd`, `code-review` skills
