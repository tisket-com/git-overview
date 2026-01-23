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

| File | Purpose |
|------|---------- |
| `operations/git.ts` | Git operations (status, commit, push) |
| `operations/files.ts` | File read/write operations |
| `tools/workspaces.ts` | Workspace tools with SSE event emission |
| `lib/github.ts` | GitHub API for repo management |
| `lib/events.ts` | Event emission for SSE |

## Shared

| File | Purpose |
|------|---------- |
| `packages/shared/src/types/` | Shared TypeScript types |
| `packages/shared/src/db/schema.ts` | Database schema including repo members |

## Key Patterns

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
