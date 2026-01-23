# Git Integration Overview

Tisket uses a hybrid client-server git architecture that enables a responsive, offline-capable experience while maintaining sync with GitHub.

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                         Browser                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ BrowserGitContext│  │  isomorphic-git │  │ LightningFS  │ │
│  │  (React Context) │──│  (Git Engine)   │──│ (IndexedDB)  │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP/SSE
┌───────────────────────────┴─────────────────────────────────┐
│                         Server                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Git Proxy     │  │   GitHub API    │  │  SSE Events  │ │
│  │  (/api/git/*)   │  │   (Fallback)    │  │ (/api/events)│ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Key Principles

1. **Client-First**: Git operations run in the browser for speed and offline capability
2. **Graceful Degradation**: Falls back to GitHub API when local operations fail
3. **Real-Time Sync**: Server-Sent Events push commits to all connected clients
4. **Optimistic Updates**: Changes appear immediately, sync happens in background

## Documentation Structure

- [Client-Side Operations](client-operations.md) - How git works in the browser
- [Repo Management](repo-management.md) - Cloning, storage, and access
- [Diff Viewing](diff-viewing.md) - Comparing file versions
- [Commit History](commit-history.md) - Timeline and version tracking
- [Real-Time Updates](realtime-updates.md) - SSE and live sync
- [Server Operations](server-operations.md) - API endpoints and fallbacks
