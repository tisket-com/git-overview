# Repository Management

How repositories are initialized and synced in the browser.

## Bootstrap via Git Clone

When a repository is first accessed, it's cloned directly from the git server:

```typescript
async bootstrap(): Promise<string | null> {
  const url = getGitUrl(this.repoSlug);

  await this.wipe();
  await ensureDir(this.fs, this.dir);

  await git.clone({
    fs: this.fs,
    http,
    dir: this.dir,
    url,
    ref: "main",
    singleBranch: true,
    depth: 50,
  });

  const commits = await git.log({ fs: this.fs, dir: this.dir, depth: 1 });
  const headSha = commits[0]?.oid || null;

  if (headSha) {
    await this.writeSyncMetadata({ remoteHead: headSha });
  }

  return headSha;
}
```

### Key Points

- Uses `git.clone()` to get real git objects with correct OIDs
- Clones with `depth: 50` to get recent history
- Single branch mode for efficiency
- Stores head SHA in sync metadata

## Storage

Each repository is stored in its own LightningFS instance backed by IndexedDB:

```typescript
const fsInstances = new Map<string, LightningFS>();

function getFS(repoSlug: string): LightningFS {
  if (!fsInstances.has(repoSlug)) {
    fsInstances.set(repoSlug, new LightningFS(`tisket-${repoSlug}`));
  }
  return fsInstances.get(repoSlug)!;
}
```

Data persists across page reloads and browser sessions.

## Sync Metadata

A `.tisket-sync` file tracks the last known remote head:

```typescript
interface SyncMetadata {
  remoteHead: string;
}

private async writeSyncMetadata(metadata: SyncMetadata): Promise<void> {
  await this.fs.promises.writeFile(
    `${this.dir}/${SYNC_FILE}`,
    JSON.stringify(metadata),
    "utf8"
  );
}
```

## Real-Time Updates via Fetch

When an SSE commit event arrives, the client fetches new objects:

```typescript
async applyCommit(commitData: CommitData): Promise<string | null> {
  // Check if commit already exists
  if (await this.commitExists(commitData.oid)) {
    await this.checkoutCommit(commitData.oid);
    return null;
  }

  // Fetch new objects and sync
  return this.fetchAndSync(commitData.oid);
}

private async fetchAndSync(targetOid: string): Promise<string | null> {
  const url = getGitUrl(this.repoSlug);

  await git.fetch({
    fs: this.fs,
    http,
    dir: this.dir,
    url,
    ref: "main",
    singleBranch: true,
  });

  await git.writeRef({
    fs: this.fs,
    dir: this.dir,
    ref: "refs/heads/main",
    value: targetOid,
    force: true,
  });

  await this.checkoutCommit(targetOid);
  await this.writeSyncMetadata({ remoteHead: targetOid });

  return targetOid;
}
```

## Git Object Integrity

By using `git.clone()` and `git.fetch()`, all git objects maintain proper integrity:

| Operation | Result |
|-----------|--------|
| `git.clone()` | Real commit/tree/blob objects with correct OIDs |
| `git.fetch()` | Fetches new objects, OIDs match upstream |
| `git.checkout()` | Updates working directory from objects |
| `readCommit(oid)` | Works for any commit in history |
| `readFileAtCommit()` | Works for any file at any commit |

## History Access

With real git history, all operations work locally:

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

async getFileHistory(filepath: string, depth: number = 50): Promise<GitCommit[]> {
  const result: GitCommit[] = [];
  const commits = await git.log({ fs: this.fs, dir: this.dir, ref: "main", depth });

  for (const commit of commits) {
    try {
      await git.readBlob({ fs: this.fs, dir: this.dir, oid: commit.oid, filepath });
      result.push({ ... });
    } catch {
      // File didn't exist at this commit
    }
  }

  return result;
}
```

## Clearing Cache

To force a fresh clone, clear IndexedDB:

1. Open DevTools > Application > IndexedDB
2. Delete databases starting with `tisket-`
3. Refresh the page