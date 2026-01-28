# Git Server Implementation Patterns

Research into how production git services handle remote operations, session isolation, and concurrency. This informs [TKT-025: Migrate MCP server from isomorphic-git to native git](../engineering.tickets/tickets/TKT-025.md).

## Overview

When building a git proxy layer (like our MCP server), there are several production-proven patterns to draw from:

| Service | Language | Approach | Notable Pattern |
|---------|----------|----------|-----------------|
| GitLab Gitaly | Go | RPC over native git | Process caching, pack-objects cache |
| Soft Serve | Go | Multi-protocol server | Hook system, SQLite metadata |
| Dugite / GitHub Desktop | Node.js | CLI wrapper | Concurrent/exclusive locks |
| Git HTTP Protocol | - | Stateless server | Client manages all state |

---

## GitLab Gitaly

[Gitaly](https://docs.gitlab.com/administration/gitaly/) is GitLab's dedicated Git RPC service. It handles all git operations for GitLab, providing high-level RPC access to repositories.

### Architecture

```
GitLab Rails ──┐
               │
gitlab-shell ──┼──► Gitaly ──► Native Git ──► Disk
               │       │
gitlab-workhorse ──┘   └──► Praefect (HA router)
```

- **RPC wrapper** - All git operations go through gRPC calls, which Gitaly translates to native git commands
- **Written in Go** - Fast, handles concurrency well
- **Praefect** - Router and transaction manager for replication and HA

### Key Patterns

**1. Process Caching**

Gitaly keeps `git cat-file --batch` processes warm for reuse across RPC calls:

```
Request 1: GetBlob("abc123") 
  → reuses existing cat-file process
  
Request 2: GetBlob("def456")
  → same process, no spawn overhead
```

**2. Pack-Objects Cache**

Caches packfiles generated during fetch operations:

- `PostUploadPack` (HTTP fetch)
- `SSHUploadPack` (SSH fetch)

This is critical for repos with many clones/fetches.

**3. Hook Execution**

Git hooks run via `gitaly-hooks` binary to ensure proper server context:

```
git-receive-pack ──► gitaly-hooks ──► actual hook logic
```

### Relevance to TKT-025

- Validates shelling out to native git (vs library)
- Process caching could optimize repeated operations
- Hook model could inform auto-commit triggers

---

## Soft Serve (Charm)

[Soft Serve](https://github.com/charmbracelet/soft-serve) is a lightweight, self-hostable git server with a TUI interface.

### Architecture

Single binary that runs multiple servers:

| Protocol | Port | Use |
|----------|------|-----|
| SSH | 23231 | Primary interface, includes TUI |
| HTTP | 23232 | Git HTTP smart protocol |
| Git daemon | 9418 | Anonymous read access |
| Stats | 23233 | Metrics |

### Key Patterns

**1. SQLite for Metadata**

- Authentication, authorization, config in SQLite
- Repository data stays in git
- Clean separation of concerns

```yaml
# config.yaml
db:
  driver: sqlite  # or postgres for scale
```

**2. Server-Side Hooks**

Supports standard git hooks with global + per-repo options:

- `pre-receive` - Validate before accepting push
- `update` - Per-ref validation
- `post-receive` - Trigger actions after push
- `post-update` - Legacy hook

**3. Auto-Create Repositories**

Push to a non-existent repo creates it (if user has write access):

```bash
git remote add origin ssh://soft-serve/new-repo.git
git push -u origin main
# Repo created automatically
```

### Relevance to TKT-025

- Hook system could inform timeout-based auto-commit (timeout as synthetic post-write hook)
- SQLite for session state is a proven pattern
- Multi-protocol support shows git operations can be protocol-agnostic

---

## Dugite / GitHub Desktop

[Dugite](https://github.com/desktop/dugite) is GitHub Desktop's git library - a Node.js wrapper around the git CLI.

### Architecture

```
GitHub Desktop (Electron)
        │
        ▼
    Dugite (Node.js)
        │
        ▼
    Bundled Git Binary
```

- **CLI wrapper** - Calls git via `GitProcess.exec()`
- **Bundles git** - Includes git binaries for Windows, macOS, Linux
- **Production-proven** - Powers GitHub Desktop for millions of users

### Key Patterns

**1. Concurrent/Exclusive Locks**

[GitHub's blog post](https://github.blog/2015-10-20-git-concurrency-in-github-desktop/) explains their concurrency model:

```
Problem: Multiple operations interleave → race conditions

Solution: Lock at "unit of work" level
- Concurrent locks: Multiple readers OK
- Exclusive locks: Single writer
```

Before implementing locks, they had race conditions. After: stable and performant.

**2. Credential Handling via GIT_ASKPASS**

```javascript
// Set env var pointing to credential helper
env.GIT_ASKPASS = '/path/to/credential-helper'

// Git calls the helper twice:
// 1. "Username for 'https://github.com':"
// 2. "Password for 'https://user@github.com':"
```

No credentials stored in process memory or command args.

**3. IGitResult Abstraction**

```typescript
interface IGitResult {
  exitCode: number
  stdout: string
  stderr: string
}

// Clean error handling
const result = await GitProcess.exec(['status'], repoPath)
if (result.exitCode !== 0) {
  // Handle error
}
```

### Relevance to TKT-025

- **Most directly applicable** - Same language (Node.js), same use case
- Concurrent/exclusive lock pattern solves session isolation
- GIT_ASKPASS for credentials is cleaner than passing tokens

---

## Git Smart HTTP Protocol

The [protocol specification](https://git-scm.com/docs/http-protocol) itself provides architectural guidance.

### Key Principles

**1. Stateless Server**

> "The Git over HTTP protocol (much like HTTP itself) is stateless from the perspective of the HTTP server side. All state MUST be retained and managed by the client process."

This enables:
- Simple round-robin load balancing
- No sticky sessions required
- Server can be restarted without losing state

**2. Simple Proxy Pattern**

Protocol v2 was designed so the HTTP handler can "simply act as a proxy":

```
Client ──► HTTP Proxy ──► Git Backend
           (stateless)    (does the work)
```

**3. Reference Implementation**

`git-http-backend` is the reference CGI:

```bash
# Apache config
ScriptAlias /git/ /usr/lib/git-core/git-http-backend/
```

It handles:
- Smart protocol negotiation
- Packfile generation
- No authentication (delegated to web server)

### Relevance to TKT-025

- Stateless model validates our session-based approach (state in worktree, not server memory)
- Proxy pattern: MCP server is a thin layer, git does the real work

---

## Recommendations for TKT-025

Based on this research:

### Git Execution: CLI Wrapper

Use `simple-git` or direct shell execution (like Dugite and Gitaly do):

```typescript
// Option 1: simple-git
import simpleGit from 'simple-git'
const git = simpleGit(repoPath)
await git.add('.')
await git.commit('message')

// Option 2: Direct execution (more control)
import { exec } from 'child_process'
await exec('git add .', { cwd: repoPath })
```

**Why:** Production-proven, full git feature set, easier debugging.

### Session Isolation: Worktrees + Locks

Combine git worktrees with Dugite-style concurrent/exclusive locks:

```typescript
class SessionManager {
  private locks = new Map<string, Lock>()
  
  async withExclusiveLock<T>(
    repoPath: string, 
    fn: () => Promise<T>
  ): Promise<T> {
    const lock = this.getLock(repoPath)
    await lock.acquireExclusive()
    try {
      return await fn()
    } finally {
      lock.release()
    }
  }
}
```

**Why:** Worktrees provide filesystem isolation; locks prevent race conditions.

### Credentials: GIT_ASKPASS

```typescript
// Create temp script that echoes the token
const askpass = createAskpassScript(token)
const env = { GIT_ASKPASS: askpass }

await git.push({ env })
```

**Why:** Secure, no tokens in command args or process memory.

### Auto-Commit: Hook-Inspired Timeout

Model the timeout as a synthetic hook:

```typescript
// After any write operation
scheduleAutoCommit(sessionId, {
  delay: 15_000,  // 15 seconds
  onTrigger: () => commitAndPush(sessionId),
  cancelOn: ['explicit_commit', 'new_write']
})
```

**Why:** Soft Serve's hook model shows this is a natural extension point.

### State Management: Stateless Server

Keep session state in:
1. Worktree on disk (git state)
2. SQLite/memory (session metadata)

Don't require:
- Sticky sessions
- Persistent connections
- Server-side session affinity

**Why:** Git HTTP protocol proves this works at scale.

---

## References

- [GitLab Gitaly Documentation](https://docs.gitlab.com/administration/gitaly/)
- [Gitaly Protocol Docs](https://gitlab-org.gitlab.io/gitaly/)
- [Soft Serve GitHub](https://github.com/charmbracelet/soft-serve)
- [Dugite GitHub](https://github.com/desktop/dugite)
- [Git Concurrency in GitHub Desktop](https://github.blog/2015-10-20-git-concurrency-in-github-desktop/)
- [Git HTTP Protocol](https://git-scm.com/docs/http-protocol)
- [Git Smart HTTP](https://git-scm.com/book/en/v2/Git-on-the-Server-Smart-HTTP)
- [simple-git npm](https://www.npmjs.com/package/simple-git)
