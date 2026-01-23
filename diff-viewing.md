# Diff Viewing System

How file changes between commits are displayed.

## Components

### DiffViewer

Dialog showing all files changed in a commit:

```typescript
interface DiffViewerProps {
  commitOid: string;
  files: Map<string, GitFileChangeType>;
  onFileSelect: (path: string) => void;
}
```

### InlineDiffRenderer

Renders diffs inline within the document view:

```typescript
interface InlineDiffRendererProps {
  filePath: string;
  diff: FileDiff;
  fromContent: string;
  toContent: string;
  isMarkdown: boolean;
}
```

## Diff State

```typescript
interface DiffModeState {
  repoSlug: string;
  filePath: string;
  fromOid: string;         // Parent commit
  toOid: string;           // Selected commit
  fromContent: string;     // Content before
  toContent: string;       // Content after
  diff: FileDiff;          // Computed diff
  commitMessage: string;
  hasChanges: boolean;
  fileExistsAtCommit: boolean;
}
```

## Diff Flow

```
1. User clicks commit in timeline
   ↓
2. viewDiff(commitOid) called
   ↓
3. Load commit metadata
   ├─ getCommitFiles(oid) → changed files
   └─ getCommitInfo(oid) → message, parent
   ↓
4. Navigate to first changed file
   ├─ Update URL with ?commit=oid
   └─ Set selectedCommitOid in context
   ↓
5. Doc viewer detects diff mode
   ↓
6. Load file diff
   ├─ readFileAtCommit(path, parentOid)
   ├─ readFileAtCommit(path, commitOid)
   └─ computeDiff(before, after)
   ↓
7. Set diffMode state
   ↓
8. InlineDiffRenderer displays changes
```

## Diff Computation

Uses the `diff` library to compute changes:

```typescript
import { structuredPatch, diffLines } from "diff";

function computeDiff(
  fromContent: string,
  toContent: string,
  fromPath: string,
  toPath: string
): FileDiff {
  const patch = structuredPatch(
    fromPath,
    toPath,
    fromContent,
    toContent
  );
  
  return {
    hunks: patch.hunks,
    additions: countAdditions(patch),
    deletions: countDeletions(patch)
  };
}
```

## Markdown-Aware Diffs

For markdown files, changes are rendered inline with styled tags:

```html
<p>This is <del>old text</del><ins>new text</ins> in a paragraph.</p>
```

The renderer:
1. Parses both versions as markdown AST
2. Walks trees in parallel, finding differences
3. Wraps changes in `<ins>` and `<del>` tags
4. Renders merged HTML

## File Change Types

```typescript
type GitFileChangeType = "added" | "modified" | "deleted" | "renamed";

const changeTypeColors = {
  added: "bg-green-500/10 border-l-2 border-green-500",
  modified: "bg-blue-500/10 border-l-2 border-blue-500",
  deleted: "bg-red-500/10 border-l-2 border-red-500",
  renamed: "bg-yellow-500/10 border-l-2 border-yellow-500"
};
```

## URL Integration

The commit OID is stored in the URL query parameter:

```
/workspace/docs/my-project/getting-started?commit=abc123
```

This enables:
- Shareable diff links
- Browser back/forward navigation
- Bookmarkable diff states
