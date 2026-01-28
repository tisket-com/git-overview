# Key Files Reference

Quick reference to the main files involved in git integration.

## Client-Side

| File | Purpose |
|------|---------- |
| `browser-git-context.tsx` | Main React context for all git state and operations |
| `workspace-context.tsx` | Workspace state with tRPC query for repos list |
| `use-browser-git.ts` | `BrowserGitRepo` class wrapping isomorphic-git |
| `use-realtime-events.ts` | SSE client hook with cache invalidation |
| `diff.ts` | Diff computation utilities |
| `inline-diff-renderer.tsx` | Renders inline diffs in document view |
| `diff-viewer.tsx` | Dialog showing commit file list |
| `file-versions.tsx` | File version history sidebar panel |
| `workspace-sidebar-live.tsx` | Sidebar with commit timeline |
| `git-ui.ts` | UI utilities (icons, colors, formatting) |

## Server-Side

| File | Purpose |
|------|---------- |
| `routes/api/git.$.ts` | Git protocol proxy |
| `routes/api/repo-bootstrap.ts` | Initial file tree API |
| `routes/api/repo-history.ts` | Commit history and file-at-commit API |
| `routes/api/events.ts` | SSE endpoint for real-time updates |
| `lib/repo-access.ts` | Access control and permissions |
| `lib/github.ts` | GitHub API client |
| `lib/repos.ts` | Repo discovery and metadata |
| `trpc/routers/repos.ts` | tRPC router for repos queries and mutations |

## MCP Server

Uses `simple-git` (native git) for all git operations. Write operations stage changes without auto-committing.

### Core Git

| File | Purpose |
|------|---------- |
| `lib/worktree.ts` | Per-user worktree isolation, staging, commit/push workflow |
| `lib/askpass.ts` | GIT_ASKPASS script management for token auth |
| `operations/git.ts` | Git tools: status, commit, push, log, commit_changes |
| `operations/repos.ts` | Repo initialization, clone, fetch, pull |

### Write Operations (stage-only)

| File | Purpose |
|------|---------- |
| `operations/files.ts` | File write/delete/rename - stages only, returns terse JSON |
| `operations/tickets.ts` | Ticket create/update - stages only, returns terse JSON |
| `operations/boards.ts` | Board create/modify - stages only, returns terse JSON |
| `operations/timelines.ts` | Timeline create/add entry - stages only, returns terse JSON |

### Infrastructure

| File | Purpose |
|------|---------- |
| `tools/workspaces.ts` | Workspace tools with SSE event emission |
| `lib/github.ts` | GitHub API for repo management |
| `lib/events.ts` | Event emission for SSE |
| `session.ts` | Session management with worktree tracking |

## Shared

| File | Purpose |
|------|---------- |
| `packages/shared/src/types/` | Shared TypeScript types |
| `packages/shared/src/db/schema.ts` | Database schema including repo members |

## Key Patterns

### MCP Write Response Format

All write operations return terse JSON with pending files:

```typescript
{
  "success": true,
  "path": "tickets/TKT-001.md",
  "bytes_written": 256,
  "pending_files": ["tickets/TKT-001.md"],
  "hint": "1 file(s) pending. Call commit_changes when done."
}
```

### Stage-Only Workflow

```typescript
// 1. Write stages the file (no commit, no push)
await mcp.write({ file_path: "doc.md", contents: "..." });

// 2. More writes accumulate in pending_files
await mcp.write({ file_path: "other.md", contents: "..." });

// 3. Explicit commit when ready
await mcp.git_commit({ message: "Update docs" });
// or
await mcp.commit_changes({ message: "Update docs" });
```

### Workspace Context

```typescript
// workspace-context.tsx
const { data: availableRepos } = trpc.repos.list.useQuery(undefined, {
  initialData: initialAvailableRepos,
  refetchOnWindowFocus: true,
  staleTime: 30_000,
});
```

### SSE Cache Invalidation

```typescript
// use-realtime-events.ts
eventSource.onopen = () => {
  queryClient.invalidateQueries({ queryKey: [["repos"]] });
};

// On workspace event
if (event.type === "workspace") {
  queryClient.invalidateQueries({ queryKey: [["repos"]] });
}
```

### MCP Event Emission

```typescript
// tools/workspaces.ts
await publishWorkspaceEvent(
  session.user.id,
  "repo_added",
  workspace.id,
  workspace.slug,
  repoName,
  "project"
);
```
