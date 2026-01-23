# Commit History & Timelines

How commit history is tracked and displayed.

## Two Types of Timelines

### Repo Timeline

All commits for the active repository, shown in the sidebar:

```typescript
const repoTimeline: GitCommit[] = [
  { oid: "abc123", message: "Update docs", timestamp: 1706000000 },
  { oid: "def456", message: "Add new page", timestamp: 1705900000 },
  // ...
];
```

Loaded when a repository becomes active. Cached per repo to prevent reloading when navigating between files.

### File Timeline

Commits affecting the current file, shown in the version history panel:

```typescript
const fileTimeline: GitCommit[] = [
  { oid: "abc123", message: "Update getting started guide", timestamp: 1706000000 },
  { oid: "xyz789", message: "Fix typo", timestamp: 1704000000 },
];
```

Loaded when the current file changes.

## Loading Commit History

### Local History

```typescript
async log(depth: number = 100): Promise<GitCommit[]> {
  const commits = await git.log({
    fs: this.fs,
    dir: this.dir,
    depth
  });
  
  return commits.map(c => ({
    oid: c.oid,
    message: c.commit.message,
    author: c.commit.author.name,
    timestamp: c.commit.author.timestamp
  }));
}
```

### GitHub History Integration

Local history may be incomplete (shallow clone). GitHub API provides full history:

```typescript
async getGitHubHistory(): Promise<GitCommit[]> {
  // Check cache (5-minute TTL)
  if (this.githubHistoryCache && !this.isCacheExpired()) {
    return this.githubHistoryCache;
  }
  
  // Fetch from API
  const response = await fetch(`/api/repo-history?repo=${this.repoSlug}`);
  const { commits } = await response.json();
  
  this.githubHistoryCache = commits;
  return commits;
}
```

### Merged History

Local and GitHub commits are merged, with duplicates removed:

```typescript
async getFullHistory(): Promise<GitCommit[]> {
  const [local, github] = await Promise.all([
    this.log(),
    this.getGitHubHistory()
  ]);
  
  // Merge and dedupe by OID
  const seen = new Set<string>();
  const merged: GitCommit[] = [];
  
  for (const commit of [...local, ...github]) {
    if (!seen.has(commit.oid)) {
      seen.add(commit.oid);
      merged.push(commit);
    }
  }
  
  // Sort by timestamp descending
  return merged.sort((a, b) => b.timestamp - a.timestamp);
}
```

## File History

Getting commits that affected a specific file:

```typescript
async getFileHistory(filepath: string, depth: number = 50): Promise<GitCommit[]> {
  const allCommits = await this.log(depth);
  const fileCommits: GitCommit[] = [];
  
  for (const commit of allCommits) {
    const files = await this.getCommitFiles(commit.oid);
    if (files.has(filepath)) {
      fileCommits.push(commit);
    }
  }
  
  return fileCommits;
}
```

## UI Components

### Sidebar Timeline

Shows repo commits with:
- Commit message (truncated)
- Relative timestamp ("2 hours ago")
- Click to view diff

### File Versions Panel

Shows file-specific history:
- Expandable list (1 by default)
- Author and timestamp
- Click to view that version's diff

## Stale-While-Revalidate

Timelines use a stale-while-revalidate pattern:

```typescript
// Show cached data immediately
if (cachedTimeline) {
  setRepoTimeline(cachedTimeline);
}

// Fetch fresh data in background
const freshTimeline = await repo.log();
setRepoTimeline(freshTimeline);
```

This prevents loading flashes when navigating.
