# AFK Issue Loop

Autonomous GitHub issue processing loop for AI coding agents.

## Language

**Worktree Registry**:
The in-memory set of git worktree paths explicitly created and tracked by the current orchestrator session.
_Avoid_: Global worktrees, existing worktree scan

**Settled Worktree**:
An isolated git worktree created for an issue that has completed its lifecycle (merged and synced to main, or closed) and is safe for teardown.
_Avoid_: Abandoned worktree, stale worktree

**Foreign Worktree**:
Any git worktree not created by the active orchestrator run, including worktrees created by other agents, previous sessions, or human developers.
_Avoid_: Orphan worktree, external branch
