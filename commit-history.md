# Commit History

How commit history is managed in the browser git repo.

## Initial Clone Depth

Repos are cloned with `depth: 50` for efficiency:

```typescript
await git.clone({
  fs: this.fs,
  http,
  dir: this.dir,
  url,
  ref: "main",
  singleBranch: true,
  depth: 50,
});
```

This provides recent history for most use cases while keeping initial load fast.

## Fetching Deeper History

When accessing commits beyond the initial depth, the client automatically fetches more:

```typescript
private async fetchDeeperHistory(): Promise<boolean> {
  try {
    const url = getGitUrl(this.repoSlug);
    await git.fetch({
      fs: this.fs,
      http,
      dir: this.dir,
      url,
      ref: "main",
      singleBranch: true,
    });
    return true;
  } catch {
    return false;
  }
}
```

This is called transparently when:
- `readFileAtCommit()` fails for an older commit
- `getCommitFiles()` fails for an older commit  
- `getCommitInfo()` fails for an older commit

## Example: Reading Older Commits

```typescript
async readFileAtCommit(filepath: string, oid: string): Promise<string | null> {
  try {
    // Try local first
    const { blob } = await git.readBlob({ ... });
    return new TextDecoder().decode(blob);
  } catch {
    // Commit not in local history - fetch more
    const fetched = await this.fetchDeeperHistory();
    if (!fetched) return null;
    
    // Retry with deeper history
    try {
      const { blob } = await git.readBlob({ ... });
      return new TextDecoder().decode(blob);
    } catch {
      return null;
    }
  }
}
```

## No API Fallbacks

All history access uses proper git operations:

| Operation | Approach |
|-----------|----------|
| Read file at commit | `git.readBlob()` with auto-fetch |
| Get commit files | `git.walk()` with auto-fetch |
| Get commit info | `git.readCommit()` with auto-fetch |
| View diff | Uses local git objects |

This ensures:
- Git object integrity (OIDs match upstream)
- Consistent behavior
- No mixed local/API data

## Timeline Display

The timeline shows commits from local git log:

```typescript
async log(depth: number = 50): Promise<GitCommit[]> {
  const commits = await git.log({ fs: this.fs, dir: this.dir, ref: "main", depth });
  return commits.map((c) => ({
    oid: c.oid,
    message: c.commit.message,
    author: c.commit.author.name,
    timestamp: c.commit.author.timestamp * 1000,
  }));
}
```

## File History

File-specific history filters commits that touched a particular file:

```typescript
async getFileHistory(filepath: string, depth: number = 50): Promise<GitCommit[]> {
  const result: GitCommit[] = [];
  const commits = await git.log({ fs: this.fs, dir: this.dir, ref: "main", depth });

  for (const commit of commits) {
    try {
      // Check if file exists at this commit
      await git.readBlob({ fs: this.fs, dir: this.dir, oid: commit.oid, filepath });
      result.push({ ... });
    } catch {
      // File didn't exist at this commit - skip
    }
  }

  return result;
}
```