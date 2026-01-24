# Git Overview

Documentation explaining how git interacts with the Tisket app.

## Contents

- [Overview](./overview.md) - High-level architecture
- [Client Operations](./client-operations.md) - Browser-side git with isomorphic-git
- [Server Operations](./server-operations.md) - MCP server git handling
- [Real-Time Updates](./realtime-updates.md) - SSE-based sync using git fetch
- [Diff Viewing](./diff-viewing.md) - Inline markdown diffs
- [Commit History](./commit-history.md) - Timeline and file history
- [Caching Strategy](./caching-strategy.md) - How data is cached
- [Repo Management](./repo-management.md) - Bootstrap and initialization
- [Key Files](./key-files.md) - Important source files

## Recent Updates

### January 24, 2026 - Git Fetch for Real-Time Sync

Updated the real-time sync mechanism to use proper `git.fetch()` instead of reconstructing git objects locally. This ensures:

- Git object OIDs match upstream exactly
- `readCommit(oid)` works correctly
- Diff viewing between commits is accurate
- History navigation is consistent

See [Real-Time Updates](./realtime-updates.md) for details.