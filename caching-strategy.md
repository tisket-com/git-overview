# Caching Strategy

How data is cached and kept fresh across the application.

## Overview

The app uses a layered caching approach:

1. **tRPC/React Query** - Server data (repos, workspaces)
2. **LightningFS** - Git repository content (IndexedDB)
3. **Local State** - UI state (React useState)

## tRPC Query Caching

Server data like repository lists use tRPC queries with built-in caching:

```typescript
const { data: availableRepos } = trpc.repos.list.useQuery(undefined, {
  initialData: serverSideRepos,    // SSR data as initial
  refetchOnWindowFocus: true,      // Refetch when tab regains focus
  staleTime: 30_000,               // Fresh for 30 seconds
});
```

### Benefits

- **Automatic deduplication** - Multiple components using same query share cache
- **Background refetch** - Updates happen without blocking UI
- **Optimistic updates** - Can update cache before server confirms
- **Built-in invalidation** - Easy to trigger refetch when needed

### Invalidation Triggers

The cache is invalidated in these scenarios:

| Trigger | Action |
|---------|--------|
| Window focus | Automatic refetch (built-in) |
| SSE reconnect | `queryClient.invalidateQueries()` |
| Workspace event | `queryClient.invalidateQueries()` |
| Manual action | `utils.repos.list.invalidate()` |

## Git Content Caching

Repository content is cached in LightningFS (IndexedDB):

```typescript
class BrowserGitRepo {
  private fs: LightningFS;  // Persists to IndexedDB
  
  async readFile(path: string): Promise<string> {
    // Reads from local cache first
    return this.fs.promises.readFile(`${this.dir}/${path}`, "utf8");
  }
}
```

### Sync Strategy

1. **On page load** - Sync with remote if needed
2. **On SSE event** - Apply commit immediately to local cache
3. **On reconnect** - Check for missed commits

```typescript
async checkRemoteAndSync(): Promise<void> {
  const syncMeta = await this.getSyncMetadata();
  const remoteHead = await this.fetchRemoteHead();
  
  if (syncMeta.remoteHead !== remoteHead) {
    await this.syncWithRemote();
  }
}
```

## Workspace Context

Workspace state uses a hybrid approach:

```typescript
function WorkspaceProvider({ initialWorkspace }) {
  // Server data via tRPC query
  const { data: repos } = trpc.repos.list.useQuery();
  
  // Local UI state
  const [currentWorkspace, setCurrentWorkspace] = useState(initialWorkspace);
  
  // Derived state
  const workspaceRepos = useMemo(() => 
    repos.filter(r => workspaceRepoSlugs.includes(r.slug)),
    [repos, workspaceRepoSlugs]
  );
}
```

### Why Hybrid?

- **Repos list** - Changes externally (MCP creates repos), needs query cache
- **Current workspace** - Changes locally (user switches), needs local state
- **Derived data** - Computed from both, uses useMemo

## Stale-While-Revalidate Pattern

Many parts of the app show cached data immediately while fetching fresh data:

```typescript
// Timeline shows cached commits immediately
// Fetches fresh commits in background
const { data: timeline, isRefetching } = trpc.repos.getCommits.useQuery(
  { repoSlug },
  { staleTime: 60_000 }
);
```

This prevents loading flashes when navigating.

## Cache Persistence

| Data Type | Storage | Persists Across |
|-----------|---------|----------------|
| Repos list | Memory (React Query) | Page reloads (with SSR) |
| Git content | IndexedDB (LightningFS) | Sessions |
| Workspace selection | URL/Cookie | Sessions |
| Diff mode | URL query param | Link sharing |

## Best Practices

1. **Use queries for server data** - Get automatic caching and invalidation
2. **Use local state for UI state** - Faster updates, no network needed
3. **Invalidate on events** - SSE events trigger cache invalidation
4. **Refetch on focus** - Catches changes made while tab was backgrounded
5. **Show stale data** - Better UX than loading spinners
