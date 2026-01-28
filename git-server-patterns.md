# Git Server Implementation Patterns

Research into how production git services handle remote operations, session isolation, and concurrency. This informs [TKT-025: Migrate MCP server from isomorphic-git to native git](../engineering.tickets/tickets/TKT-025.md).

## Overview

When building a git proxy layer (like our MCP server), there are several production-proven patterns to draw from:

| Service | Language | Approach | Notable Pattern |
|---------|----------|----------|-----------------|
| GitLab Gitaly | Go | RPC over native git | Process caching, pack-objects cache |
| Soft Serve | Go | Multi-protocol server | Hook system, SQLite metadata |
| simple-git | Node.js | CLI wrapper + scheduler | Concurrency-limited task queue |
| Dugite / GitHub Desktop | Node.js | CLI wrapper | Request deduplication |
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

## simple-git (Node.js)

[simple-git](https://github.com/steveukx/git-js) is a mature Node.js wrapper around the git CLI with built-in task scheduling.

### Architecture

```
Your Code
    │
    ▼
SimpleGit API ──► GitExecutorChain ──► Scheduler ──► spawn()
                        │
                        └──► TasksPendingQueue (tracking)
```

### Key Patterns (from source code analysis)

**1. Concurrency-Limited Scheduler**

The `Scheduler` class limits concurrent git processes:

```typescript
// simple-git/src/lib/runners/scheduler.ts
export class Scheduler {
  private pending: ScheduledTask[] = [];
  private running: ScheduledTask[] = [];

  constructor(private concurrency = 2) {}

  private schedule() {
    // Only start new task if under concurrency limit
    if (!this.pending.length || this.running.length >= this.concurrency) {
      return;
    }

    const task = this.pending.shift()!;
    this.running.push(task);
    
    task.done(() => {
      remove(this.running, task);
      this.schedule();  // Try next task
    });
  }

  next(): Promise<ScheduleCompleteCallback> {
    const task = createScheduledTask();
    this.pending.push(task);
    this.schedule();
    return task.promise;
  }
}
```

Default concurrency is 2, configurable via `maxConcurrentProcesses`.

**2. Promise Chain for Sequential Execution**

Each executor chains tasks sequentially within its context:

```typescript
// simple-git/src/lib/runners/git-executor-chain.ts
export class GitExecutorChain {
  private _chain: Promise<any> = Promise.resolve();
  private _queue = new TasksPendingQueue();

  push<R>(task: SimpleGitTask<R>): Promise<R> {
    this._queue.push(task);
    // Chain tasks sequentially
    return (this._chain = this._chain.then(() => this.attemptTask(task)));
  }
}
```

**3. Fatal Error Handling with Queue Purge**

When a task fails fatally, the entire queue is purged:

```typescript
private onFatalException<R>(task: SimpleGitTask<R>, e: Error) {
  const gitError = e instanceof GitError 
    ? Object.assign(e, { task }) 
    : new GitError(task, String(e));

  // Reset chain and purge queue
  this._chain = Promise.resolve();
  this._queue.fatal(gitError);

  return gitError;
}

// In TasksPendingQueue
fatal(err: GitError) {
  for (const [task, { logger }] of this._queue.entries()) {
    logger.info(`Fatal exception, queue purged`);
    this.complete(task);
  }
}
```

**4. Plugin Architecture**

Extensible via plugins for spawn customization:

- `spawn.binary` - Custom git binary path
- `spawn.args` - Modify arguments
- `spawn.options` - Customize spawn options
- `spawn.before` / `spawn.after` - Lifecycle hooks
- `task.error` - Custom error handling

```typescript
const git = simpleGit({
  maxConcurrentProcesses: 6,
  config: ['core.autocrlf=false']
});
```

### Relevance to TKT-025

- **Ready-made concurrency control** - No need to build our own scheduler
- **Error boundary pattern** - Fatal errors don't cascade
- **Plugin system** - Can inject custom behavior (e.g., logging, auth)

---

## Dugite / GitHub Desktop

[Dugite](https://github.com/desktop/dugite) is GitHub Desktop's git library - a thin Node.js wrapper around the git CLI.

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

- **CLI wrapper** - Calls git via `exec()` from Node's child_process
- **Bundles git** - Includes git binaries for Windows, macOS, Linux
- **Production-proven** - Powers GitHub Desktop for millions of users

### Source Code Analysis

**Dugite itself is minimal** - just a wrapper around `execFile`:

```typescript
// dugite/lib/exec.ts
export function exec(
  args: string[],
  path: string,
  options?: IGitExecutionOptions
): Promise<IGitResult> {
  const { env, gitLocation } = setupEnvironment(options?.env ?? {});

  return new Promise((resolve, reject) => {
    const cp = execFile(gitLocation, args, {
      cwd: path,
      env,
      encoding: options?.encoding ?? 'utf8',
      maxBuffer: options?.maxBuffer ?? Infinity,
    }, (err, stdout, stderr) => {
      const exitCode = typeof err?.code === 'number' ? err.code : 0;
      resolve({ stdout, stderr, exitCode });
    });
  });
}
```

**Concurrency is handled at the application level** in GitHub Desktop:

```typescript
// GitHub Desktop's git-store.ts
private readonly requestsInFight = new Set<string>();

public async loadCommitBatch(commitish: string, skip: number) {
  const requestKey = `commits-${commitish}-${skip}`;
  
  // Deduplicate concurrent requests
  if (this.requestsInFight.has(requestKey)) {
    return;
  }
  
  this.requestsInFight.add(requestKey);
  try {
    // ... do work ...
  } finally {
    this.requestsInFight.delete(requestKey);
  }
}
```

**Note:** The [2015 blog post](https://github.blog/2015-10-20-git-concurrency-in-github-desktop/) about concurrent/exclusive locks was for the original C#/Objective-C codebase. The modern Electron version uses simpler patterns:

1. **Request deduplication** via `Set` tracking
2. **Error boundaries** via `performFailableOperation` wrapper
3. **Single-threaded JS** - Electron's main process naturally serializes operations

### Key Patterns

**1. Credential Handling via GIT_ASKPASS**

```javascript
// Set env var pointing to credential helper
env.GIT_ASKPASS = '/path/to/credential-helper'

// Git calls the helper twice:
// 1. "Username for 'https://github.com':"
// 2. "Password for 'https://user@github.com':"
```

No credentials stored in process memory or command args.

**2. IGitResult Abstraction**

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

- Shows that simple patterns work at scale (millions of users)
- GIT_ASKPASS for credentials is cleaner than passing tokens
- Request deduplication is simpler than formal locks for many use cases

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

### Git Execution: Use simple-git

simple-git provides everything we need out of the box:

```typescript
import simpleGit, { SimpleGit } from 'simple-git';

const git: SimpleGit = simpleGit(repoPath, {
  maxConcurrentProcesses: 4,  // Limit concurrent git processes
});

// Operations are automatically queued and rate-limited
await git.add('.');
await git.commit('message');
await git.push();
```

**Why:** Built-in scheduler, error handling, plugin system. Production-proven.

### Session Isolation: Worktrees

Each session gets its own worktree:

```typescript
async function createSessionWorktree(repoPath: string, sessionId: string) {
  const git = simpleGit(repoPath);
  const worktreePath = `/tmp/sessions/${sessionId}`;
  const branchName = `session/${sessionId}`;
  
  await git.raw(['worktree', 'add', worktreePath, '-b', branchName]);
  
  return simpleGit(worktreePath);
}
```

**Why:** Filesystem-level isolation, no lock contention between sessions.

### Credentials: GIT_ASKPASS

```typescript
import { writeFileSync, chmodSync } from 'fs';
import { tmpdir } from 'os';
import { join } from 'path';

function createAskpassScript(token: string): string {
  const script = join(tmpdir(), `askpass-${Date.now()}.sh`);
  writeFileSync(script, `#!/bin/sh\necho "${token}"\n`);
  chmodSync(script, 0o700);
  return script;
}

// Use with simple-git
const git = simpleGit(repoPath, {
  config: [`credential.helper=`],  // Disable other helpers
});

await git.env('GIT_ASKPASS', createAskpassScript(token)).push();
```

**Why:** Secure, no tokens in command args or process memory.

### Auto-Commit: Timeout with Debounce

```typescript
class SessionCommitManager {
  private timers = new Map<string, NodeJS.Timeout>();
  
  scheduleCommit(sessionId: string, delay = 15_000) {
    // Cancel existing timer
    const existing = this.timers.get(sessionId);
    if (existing) clearTimeout(existing);
    
    // Schedule new timer
    const timer = setTimeout(() => {
      this.commitAndPush(sessionId);
      this.timers.delete(sessionId);
    }, delay);
    
    this.timers.set(sessionId, timer);
  }
  
  cancelScheduledCommit(sessionId: string) {
    const timer = this.timers.get(sessionId);
    if (timer) {
      clearTimeout(timer);
      this.timers.delete(sessionId);
    }
  }
}
```

**Why:** Simple, debounced, matches observed MCP request patterns.

### State Management: Stateless Server

Keep session state in:
1. Worktree on disk (git state)
2. In-memory Map (session metadata, timers)

Don't require:
- Sticky sessions
- Persistent connections
- Server-side session affinity

**Why:** Git HTTP protocol proves this works at scale.

---

## Implementation Comparison

| Aspect | isomorphic-git (current) | simple-git (proposed) |
|--------|-------------------------|----------------------|
| Concurrency | None (single-threaded) | Built-in scheduler |
| Worktrees | Not supported | Full support |
| Error handling | Manual | Built-in with queue purge |
| Hooks | Not applicable | Plugin system |
| Bundle size | ~200KB | ~50KB (wrapper only) |
| Git binary | Bundled JS implementation | System git required |

---

## References

- [GitLab Gitaly Documentation](https://docs.gitlab.com/administration/gitaly/)
- [Gitaly Protocol Docs](https://gitlab-org.gitlab.io/gitaly/)
- [Soft Serve GitHub](https://github.com/charmbracelet/soft-serve)
- [simple-git GitHub](https://github.com/steveukx/git-js)
- [Dugite GitHub](https://github.com/desktop/dugite)
- [Git Concurrency in GitHub Desktop (2015)](https://github.blog/2015-10-20-git-concurrency-in-github-desktop/)
- [Git HTTP Protocol](https://git-scm.com/docs/http-protocol)
- [Git Smart HTTP](https://git-scm.com/book/en/v2/Git-on-the-Server-Smart-HTTP)
