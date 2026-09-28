# Scoped Worktree Cleanup with Ownership Isolation

During multi-agent and parallel issue runs, git worktrees (`.worktrees/issue-<N>`) accumulate on disk. We decided to clean up settled issue worktrees (in Phase 2 upon squash-merge, with a safety wrap-up in Phase 3) restricted strictly to worktrees explicitly tracked in the current session's Worktree Registry. Blocked or interrupted worktrees are preserved for resumption, and worktrees created by other agents or users are never touched.
