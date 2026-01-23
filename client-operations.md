# Client-Side Git Operations

All primary git operations run in the browser using isomorphic-git, providing fast, offline-capable access to repositories.

## Core Components

### BrowserGitContext

The central React context that manages all client-side git state:

```typescript
interface BrowserGitContextValue {
  // Repo Management
  ensureRepo: (slug: string) => Promise<RepoState>;
  getRepoVersion: (slug: string) => number;
  
  // File Operations
  readFile: (repoSlug: string, path: string) => Promise<string>;
  readFileAtCommit: (repoSlug: string, path: string, oid: string) => Promise<string>;
  
  // Timeline
  fileTimeline: GitCommit[];
  repoTimeline: GitCommit[];
  
  // Diff Mode
  diffMode: DiffModeState | null;
  viewDiff: (commitOid: string) => void;
  clearDiffMode: () => void;
}
```

### BrowserGitRepo Class

Wraps isomorphic-git for each repository:

```typescript
class BrowserGitRepo {
  private fs: LightningFS;     // In-browser file system
  private dir: string;          // Repo directory path
  private repoSlug: string;     // Repository identifier
  
  async initialize(): Promise<void>;
  async readFile(path: string): Promise<string>;
  async readFileAtCommit(path: string, oid: string): Promise<string>;
  async log(depth?: number): Promise<GitCommit[]>;
  async getCommitFiles(oid: string): Promise<Map<string, ChangeType>>;
  async applyCommit(commitData: CommitEvent): Promise<void>;
}
```

## Storage Architecture

Each repository gets its own LightningFS instance backed by IndexedDB:

```
IndexedDB
├── tisket-project-docs/     # LightningFS for project-docs repo
├── tisket-api-reference/    # LightningFS for api-reference repo
└── tisket-engineering.tickets/  # LightningFS for tickets repo
```

Data persists across page reloads and browser sessions.

## Version Subscription Pattern

To avoid unnecessary re-renders, the context uses a version subscription pattern:

```typescript
// Components subscribe to specific repo versions
const version = useGitRepoVersion(repoSlug);

// Only re-renders when that repo's version changes
useEffect(() => {
  loadFileContent();
}, [version]);
```

This prevents all components from re-rendering when any repo updates.

## Key Hooks

### useGitFile

Reads a file from a repository:

```typescript
const { content, isLoading, error } = useGitFile(repoSlug, filePath);
```

### useBrowserGitContext

Access the full context:

```typescript
const { viewDiff, diffMode, fileTimeline } = useBrowserGitContext();
```
