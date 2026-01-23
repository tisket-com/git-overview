# Repository Management

How repositories are cloned, stored, and accessed in the browser.

## Initialization Flow

When a user navigates to a workspace, repositories are initialized:

```
1. User navigates to workspace
   ↓
2. BrowserGitProvider mounts
   ↓
3. ensureRepo(repoSlug) called for each repo
   ↓
4. BrowserGitRepo.initialize()
   ├─ Check for existing repo in LightningFS
   ├─ If exists: checkRemoteAndSync()
   └─ If not: bootstrap()
   ↓
5. Repo ready → UI updates
```

## Cloning Methods

### Primary: Git Clone

```typescript
await git.clone({
  fs: this.fs,
  http: http,
  dir: this.dir,
  url: `/api/git/${repoSlug}`,  // Proxied to MCP server
  depth: 1,
  singleBranch: true,
  ref: "main"
});
```

The server proxies git protocol requests to the MCP server, which has GitHub credentials.

### Fallback: API Bootstrap

If git clone fails (e.g., CORS issues, network problems):

```typescript
// Fetch file tree from API
const response = await fetch(`/api/repo-bootstrap?repo=${repoSlug}`);
const { files } = await response.json();

// Create repo manually
await git.init({ fs: this.fs, dir: this.dir });

// Add all files
for (const file of files) {
  await this.fs.promises.writeFile(
    `${this.dir}/${file.path}`,
    file.content
  );
  await git.add({ fs: this.fs, dir: this.dir, filepath: file.path });
}

// Create initial commit
await git.commit({
  fs: this.fs,
  dir: this.dir,
  message: "Initial commit (bootstrapped)",
  author: { name: "Tisket", email: "system@tisket.com" }
});
```

## Sync Metadata

Each repo stores sync state in a `.tisket-sync` file:

```json
{
  "remoteHead": "abc123...",
  "appliedCommits": ["def456...", "ghi789..."]
}
```

This tracks:
- The last known remote HEAD commit
- Commits applied via SSE (to avoid duplicates)

## Repo State

```typescript
interface RepoState {
  repo: BrowserGitRepo;    // The repo instance
  head: string | null;     // Current HEAD commit
  isReady: boolean;        // Initialization complete
  isCloning: boolean;      // Currently cloning
}
```

Repos are stored in a Map keyed by slug:

```typescript
const repos = new Map<string, RepoState>();
```

## Active Repo Persistence

The `activeRepoSlug` persists during navigation to prevent timeline reloads:

```typescript
// When navigating between files in the same repo,
// the timeline doesn't reload
useEffect(() => {
  if (currentFile?.repoSlug && currentFile.repoSlug !== activeRepoSlug) {
    setActiveRepoSlug(currentFile.repoSlug);
  }
}, [currentFile?.repoSlug]);
```
