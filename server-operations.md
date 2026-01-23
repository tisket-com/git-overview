# Server-Side Operations

API endpoints that support the client-side git experience.

## Git Proxy

**Endpoint:** `/api/git/*`

Proxies git protocol requests to the MCP server:

```typescript
// client/src/routes/api/git.$.ts
export async function loader({ request, params }: LoaderFunctionArgs) {
  const gitPath = params["*"];
  const mcpUrl = `${MCP_SERVER_URL}/git/${gitPath}`;
  
  return fetch(mcpUrl, {
    method: request.method,
    headers: {
      ...request.headers,
      Authorization: `Bearer ${session.token}`
    },
    body: request.body
  });
}
```

Used during:
- Initial clone
- Fetch operations
- Push operations (via MCP)

## Repo Bootstrap

**Endpoint:** `/api/repo-bootstrap`

Provides initial file tree when git clone fails:

```typescript
// Request
GET /api/repo-bootstrap?repo=my-project

// Response
{
  "files": [
    { "path": "README.md", "content": "# My Project\n..." },
    { "path": "getting-started.md", "content": "..." }
  ],
  "headCommit": "abc123..."
}
```

Fetches from GitHub API:

```typescript
async function getRepoFiles(repoSlug: string): Promise<FileTree> {
  const octokit = new Octokit({ auth: GITHUB_TOKEN });
  
  // Get tree recursively
  const { data } = await octokit.git.getTree({
    owner: ORG_NAME,
    repo: repoSlug,
    tree_sha: "HEAD",
    recursive: "true"
  });
  
  // Fetch content for each file
  const files = await Promise.all(
    data.tree
      .filter(item => item.type === "blob")
      .map(async item => ({
        path: item.path,
        content: await getFileContent(repoSlug, item.path)
      }))
  );
  
  return { files };
}
```

## Repo History

**Endpoint:** `/api/repo-history`

Provides commit history and file content at specific commits:

```typescript
// Get commit history
GET /api/repo-history?repo=my-project

// Response
{
  "commits": [
    {
      "oid": "abc123",
      "message": "Update docs",
      "author": "Jane Doe",
      "timestamp": 1706000000
    }
  ]
}

// Get file at commit
GET /api/repo-history?repo=my-project&path=README.md&sha=abc123

// Response
{
  "content": "# My Project\n..."
}
```

## Events Endpoint

**Endpoint:** `/api/events`

Server-Sent Events for real-time updates:

```typescript
export async function loader({ request }: LoaderFunctionArgs) {
  const stream = new ReadableStream({
    start(controller) {
      // Subscribe to Redis pub/sub
      const subscriber = redis.duplicate();
      subscriber.subscribe("workspace-events");
      
      subscriber.on("message", (channel, message) => {
        const event = JSON.parse(message);
        controller.enqueue(`data: ${JSON.stringify(event)}\n\n`);
      });
      
      // Handle disconnect
      request.signal.addEventListener("abort", () => {
        subscriber.unsubscribe();
        subscriber.quit();
      });
    }
  });
  
  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive"
    }
  });
}
```

## Access Control

All endpoints check permissions:

```typescript
// lib/repo-access.ts
async function checkRepoAccess(
  userId: string,
  repoSlug: string,
  requiredLevel: "read" | "write" | "admin"
): Promise<boolean> {
  const membership = await db.query.repoMembers.findFirst({
    where: and(
      eq(repoMembers.userId, userId),
      eq(repoMembers.repoSlug, repoSlug)
    )
  });
  
  if (!membership) return false;
  
  const levels = ["read", "write", "developer", "maintain", "admin"];
  return levels.indexOf(membership.role) >= levels.indexOf(requiredLevel);
}
```

## Error Handling

Servers return structured errors:

```typescript
{
  "error": "REPO_NOT_FOUND",
  "message": "Repository 'my-project' not found",
  "status": 404
}
```

Clients fall back gracefully when server operations fail.
