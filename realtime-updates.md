# Real-Time Updates

How changes sync across clients using Server-Sent Events.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  MCP Server  │────▶│  Web Server  │────▶│   Browser    │
│  (commits)   │     │   (SSE)      │     │  (fetches)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

1. MCP server creates a commit and pushes to GitHub
2. Server emits SSE event with commit metadata
3. Browser receives event and fetches git objects from server
4. Working directory is updated via git checkout

## Event Types

```typescript
type RealtimeEvent = 
  | CommitEvent       // New commit pushed
  | WorkspaceEvent    // Workspace updated
  | RepoEvent;        // Repo added/removed

interface CommitEvent {
  type: "commit";
  repoSlug: string;
  oid: string;
  message: string;
  author: { name: string; email: string };
  timestamp: number;
  files: Array<{
    path: string;
    content: string | null;
    changeType: GitFileChangeType;
  }>;
  tree: string;
  parent: string[];
  gitAuthor: GitAuthor;
  gitCommitter: GitAuthor;
}

interface WorkspaceEvent {
  type: "workspace";
  action: "repo_added" | "repo_removed" | "updated";
  workspaceId: string;
  workspaceSlug: string;
  repoSlug?: string;
  repoType?: "project" | "team";
}
```

## Client-Side Handling

### useRealtimeEvents Hook

```typescript
function useRealtimeEvents() {
  const queryClient = useQueryClient();
  
  useEffect(() => {
    const eventSource = new EventSource("/api/events");
    
    eventSource.onopen = () => {
      queryClient.invalidateQueries({ queryKey: [["repos"]] });
    };
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      handleEvent(data);
    };
    
    return () => eventSource.close();
  }, []);
}
```

### Cache Invalidation on Events

When workspace events arrive, the tRPC cache is invalidated:

```typescript
function handleEvents(events: TisketEvent[]) {
  for (const event of events) {
    if (event.type === "commit") {
      await onCommit(event);
    } else if (event.type === "workspace") {
      queryClient.invalidateQueries({ queryKey: [["repos"]] });
    }
  }
}
```

## Applying Commits (Git Fetch Approach)

When an SSE commit event arrives, the client uses proper git fetch to sync:

```typescript
async applyCommit(commitData: CommitData): Promise<string | null> {
  // Check if commit already exists locally
  if (await this.commitExists(commitData.oid)) {
    await this.checkoutCommit(commitData.oid);
    return null;
  }

  // Fetch git objects from server
  const newHead = await this.fetchAndSync(commitData.oid);
  return newHead;
}

private async fetchAndSync(targetOid: string): Promise<string | null> {
  const url = getGitUrl(this.repoSlug);

  // Fetch actual git objects (blobs, trees, commits)
  await git.fetch({
    fs: this.fs,
    http,
    dir: this.dir,
    url,
    ref: "main",
    singleBranch: true,
  });

  // Update local ref to point to new commit
  await git.writeRef({
    fs: this.fs,
    dir: this.dir,
    ref: "refs/heads/main",
    value: targetOid,
    force: true,
  });

  // Update working directory
  await this.checkoutCommit(targetOid);

  await this.writeSyncMetadata({ remoteHead: targetOid });
  return targetOid;
}

private async checkoutCommit(oid: string): Promise<void> {
  await git.checkout({
    fs: this.fs,
    dir: this.dir,
    ref: oid,
    force: true,
  });
}
```

### Why Git Fetch?

The previous approach tried to reconstruct git objects locally from file content sent via SSE. This caused OID mismatches because:

- Blob OIDs depend on exact byte content
- Tree OIDs depend on exact structure and child OIDs
- Commit OIDs depend on tree, parents, author, committer, message

If any detail differed, local OIDs wouldn't match upstream, breaking:
- `readCommit(oid)` lookups
- Diff viewing between commits
- History navigation

The git fetch approach downloads the actual git objects from the server, preserving exact OIDs.

### Performance

Git fetch is efficient for incremental updates:
- Only downloads objects not already present locally
- Uses delta compression
- For a single-file commit: ~1-5KB transfer
- Mostly network latency (~100-500ms)

## Handling Missed Events

SSE only delivers events while connected. To handle missed events:

### 1. Refetch on Window Focus

```typescript
const { data: availableRepos } = trpc.repos.list.useQuery(undefined, {
  refetchOnWindowFocus: true,
  staleTime: 30_000,
});
```

### 2. Refetch on SSE Reconnect

```typescript
eventSource.onopen = () => {
  queryClient.invalidateQueries({ queryKey: [["repos"]] });
};
```

### 3. Bootstrap on Diff Exit

When exiting diff view, the repo is refreshed to catch any missed updates:

```typescript
const clearDiffMode = useCallback(() => {
  // ... clear diff state ...
  
  if (repoSlugToRefresh) {
    const repoState = reposRef.current.get(repoSlugToRefresh);
    if (repoState?.repo) {
      repoState.repo.bootstrap().then((newHead) => {
        if (newHead) {
          repoState.head = newHead;
          incrementVersion();
        }
      });
    }
  }
}, [navigate, selectedCommitRepoSlug]);
```

### 4. Fallback to Bootstrap

If git fetch fails for any reason, the system falls back to a full bootstrap:

```typescript
async applyCommit(commitData: CommitData): Promise<string | null> {
  try {
    const newHead = await this.fetchAndSync(commitData.oid);
    return newHead;
  } catch (e) {
    console.error("Apply commit error:", e);
    return this.bootstrap();  // Full resync as fallback
  }
}
```

## Recently Changed Files

After applying a commit, files are marked as recently changed for UI highlighting:

```typescript
setRecentlyChangedFiles(prev => {
  const next = new Map(prev);
  for (const file of commit.files) {
    const key = `${commit.repoSlug}:${file.path}`;
    next.set(key, {
      changeType: file.changeType ?? "modified",
      timestamp: Date.now(),
      commitOid: commit.oid,
    });
  }
  return next;
});
```

## Timeline Updates

When a commit arrives, timelines are updated:

```typescript
function handleCommit(event: CommitEvent) {
  // Add to repo timeline
  setRepoTimeline(prev => [
    { oid: event.oid, message: event.message, author: event.author.name, timestamp: event.timestamp },
    ...prev
  ]);
  
  // Update file timeline if current file was affected
  if (currentFile && event.files.some(f => f.path === currentFile.filePath)) {
    setFileTimeline(prev => [/* new commit */, ...prev]);
  }
  
  incrementVersion();  // Trigger UI refresh
}
```

## Git Object Integrity

With the fetch-based approach, git objects maintain proper integrity:

| Object | OID Matches Upstream |
|--------|---------------------|
| Blobs | Yes (fetched) |
| Trees | Yes (fetched) |
| Commits | Yes (fetched) |

This enables:
- Accurate `readCommit(oid)` lookups
- Proper diff viewing between any commits
- Consistent history navigation
- File content at any commit via `readFileAtCommit()`