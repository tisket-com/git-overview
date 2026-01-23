# Real-Time Updates

How changes sync across clients using Server-Sent Events.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  MCP Server  │────▶│  Web Server  │────▶│   Browser    │
│  (commits)   │     │   (SSE)      │     │  (applies)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

1. MCP server creates a commit and pushes to GitHub
2. Server emits SSE event with commit data
3. Browser receives event and applies commit locally

## Event Types

```typescript
type RealtimeEvent = 
  | CommitEvent       // New commit pushed
  | WorkspaceEvent    // Workspace updated
  | RepoEvent;        // Repo added/removed

interface CommitEvent {
  type: "commit";
  repoSlug: string;
  commit: {
    oid: string;
    message: string;
    author: string;
    timestamp: number;
    files: Array<{
      path: string;
      content: string | null;  // null = deleted
      changeType: GitFileChangeType;
    }>;
  };
}
```

## Client-Side Handling

### useRealtimeEvents Hook

```typescript
function useRealtimeEvents() {
  useEffect(() => {
    const eventSource = new EventSource("/api/events");
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      handleEvent(data);
    };
    
    return () => eventSource.close();
  }, []);
}
```

### Applying Commits

```typescript
async applyCommit(commitData: CommitEvent): Promise<void> {
  // Check if already applied
  if (this.appliedCommits.has(commitData.commit.oid)) {
    return;
  }
  
  // Ensure local branch exists
  await this.ensureLocalBranch();
  
  // Write files to LightningFS
  for (const file of commitData.commit.files) {
    if (file.content === null) {
      await this.fs.promises.unlink(`${this.dir}/${file.path}`);
    } else {
      await this.fs.promises.writeFile(
        `${this.dir}/${file.path}`,
        file.content
      );
    }
  }
  
  // Build git tree and create commit
  await git.add({ fs: this.fs, dir: this.dir, filepath: "." });
  await git.commit({
    fs: this.fs,
    dir: this.dir,
    message: commitData.commit.message,
    author: {
      name: commitData.commit.author,
      email: "system@tisket.com"
    }
  });
  
  // Track as applied
  this.appliedCommits.add(commitData.commit.oid);
  await this.saveSyncMetadata();
  
  // Trigger UI update
  this.incrementVersion();
}
```

## Recently Changed Files

After applying a commit, files are marked as recently changed:

```typescript
const recentlyChangedFiles = new Map<string, number>();

function handleCommit(event: CommitEvent) {
  for (const file of event.commit.files) {
    const key = `${event.repoSlug}:${file.path}`;
    recentlyChangedFiles.set(key, Date.now());
  }
  
  // Clear after 5 seconds
  setTimeout(() => {
    for (const file of event.commit.files) {
      recentlyChangedFiles.delete(`${event.repoSlug}:${file.path}`);
    }
  }, 5000);
}
```

This enables UI highlighting of recently changed files in the sidebar.

## Timeline Updates

When a commit arrives, timelines are updated:

```typescript
function handleCommit(event: CommitEvent) {
  // Add to repo timeline
  setRepoTimeline(prev => [
    {
      oid: event.commit.oid,
      message: event.commit.message,
      author: event.commit.author,
      timestamp: event.commit.timestamp
    },
    ...prev
  ]);
  
  // Update file timeline if current file was affected
  if (currentFile && event.commit.files.some(f => f.path === currentFile.filePath)) {
    setFileTimeline(prev => [/* new commit */, ...prev]);
  }
}
```

## Branch Strategy

Applied commits go to a "local" branch:

```
main (from clone)
  └── local (SSE commits applied here)
```

This separates upstream history from locally-applied changes.
