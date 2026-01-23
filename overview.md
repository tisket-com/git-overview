# Git Integration Overview

Tisket uses a hybrid client-server git architecture that enables a responsive, offline-capable experience while maintaining sync with GitHub.

> **Last updated:** January 2026 - Added tRPC caching layer for automatic cache invalidation

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                         Browser                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ BrowserGitContext│  │  isomorphic-git │  │ LightningFS  │ │
│  │  (React Context) │──│  (Git Engine)   │──│ (IndexedDB)  │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│  ┌─────────────────┐                                           │
│  │ tRPC/React Query│  (Server data caching)                   │
│  └─────────────────┘                                           │
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
5. **Smart Caching**: tRPC/React Query for server data, LightningFS for git content

## Documentation Structure

- [Client-Side Operations](client-operations.md) - How git works in the browser
- [Repo Management](repo-management.md) - Cloning, storage, and access
- [Diff Viewing](diff-viewing.md) - Comparing file versions
- [Commit History](commit-history.md) - Timeline and version tracking
- [Real-Time Updates](realtime-updates.md) - SSE and live sync
- [Caching Strategy](caching-strategy.md) - How data stays fresh
- [Server Operations](server-operations.md) - API endpoints and fallbacks
- [Key Files](key-files.md) - Quick reference to source files
