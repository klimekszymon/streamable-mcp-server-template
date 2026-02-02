# MCP Streamable HTTP Server Template

## What is this?

A template for building MCP servers. Clone it, strip what you don't need, wire your API client, define tools. It's designed to be readable and easy to build on.

Ships with dual-runtime support (Node.js and Cloudflare Workers from the same codebase), five auth strategies, encrypted token storage, and pretty much everything the latest MCP spec supports.

## What is MCP?

Model Context Protocol is a JSON-RPC 2.0 wire protocol where servers expose typed capabilities (tools for actions, resources for data, prompts for templates) and clients (IDEs, agents, chat apps) invoke them based on LLM decisions.

Neither side implements the other's logic: servers know nothing about which LLM uses them, clients know nothing about how tools work internally. This decoupling solves the N×M integration problem. One server serves any compliant client, one client consumes any compliant server.

## What's supported?

| Feature | Node.js | Workers | Notes |
|---------|---------|---------|-------|
| Tools (list, call) | ✅ | ✅ | Core capability, both runtimes |
| Resources (list, read, templates) | ✅ | ✅ | Static and dynamic resources |
| Prompts (list, get) | ✅ | ✅ | Template-based prompt generation |
| Progress notifications | ✅ | ✅ | Long-running tool feedback |
| Cancellation | ✅ | ✅ | AbortSignal-based |
| Pagination | ✅ | ✅ | Cursor-based for large lists |
| Logging | ✅ | ✅ | Server→client log messages |
| Sampling (server→client LLM) | ✅ | ❌ | Requires persistent SSE stream |
| Elicitation (user input) | ✅ | ❌ | Requires persistent SSE stream |
| Roots (filesystem access) | ✅ | ❌ | Requires client capability check |

Protocol versions supported: `2025-11-25`, `2025-06-18`, `2025-03-26`, `2024-11-05`.

## Getting started

**First, generate an encryption key (you'll need this for both runtimes):**
```bash
openssl rand -base64 32 | tr -d '=' | tr '+/' '-_'
```

### Node.js

```bash
bun install
cp .env.example .env          # Configure PROVIDER_*, AUTH_*, OAUTH_* vars
                              # Set RS_TOKENS_ENC_KEY with generated key
bun dev                       # MCP: localhost:3000/mcp, OAuth: localhost:3001
```

### Cloudflare Workers

```bash
bun install
wrangler kv:namespace create TOKENS                    # Note the ID
# Update wrangler.toml with KV namespace ID

wrangler secret put PROVIDER_CLIENT_ID
wrangler secret put PROVIDER_CLIENT_SECRET
wrangler secret put RS_TOKENS_ENC_KEY                  # Paste generated key

wrangler dev                  # Local: localhost:8787/mcp
wrangler deploy               # Production: your-worker.workers.dev/mcp
```

## Server endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/mcp` | POST, GET, DELETE | MCP protocol (JSON-RPC) |
| `/health` | GET | Health check + readiness |
| `/.well-known/oauth-authorization-server` | GET | OAuth AS metadata |
| `/.well-known/oauth-protected-resource` | GET | Protected resource metadata |
| `/authorize` | GET | Start OAuth flow |
| `/oauth/callback` | GET | Provider redirect target |
| `/token` | POST | Token exchange |
| `/register` | POST | Dynamic client registration |
| `/revoke` | POST | Token revocation |

Discovery endpoints also available under `/mcp/.well-known/*` prefix.

## Node.js vs Cloudflare Workers

The template produces two runtimes from the same codebase. Here's what you need to know:

**Node.js (Hono + @hono/node-server)**
- Entry: `src/index.ts`
- Transport: SDK's `StreamableHTTPServerTransport`
- Sessions: `MemorySessionStore` (default) or `SqliteSessionStore` for persistence
- Full MCP features including bidirectional requests (sampling, elicitation, roots)
- Local development: `bun dev`

**Cloudflare Workers**
- Entry: `src/worker.ts`
- Transport: Custom JSON-RPC dispatcher (`shared/mcp/dispatcher.ts`)
- Sessions: `KvSessionStore` with memory fallback (persists across requests)
- Request→response only; no server-initiated messages
- Deploy: `wrangler deploy`

**Shared code** lives in `src/shared/` (tools, storage interfaces, OAuth flow, utilities). Runtime-specific adapters live in `src/adapters/http-hono/` and `src/adapters/http-workers/`.

**When to use which:**
- Node.js: Local development, full MCP features, self-hosted servers
- Workers: Production deployment, global edge, simple tool wrappers

## Authorization

### Naming conventions (important!)

Use **generic `PROVIDER_*` names**, not service-specific names. This keeps the template portable and configuration consistent across all MCP servers.

| ✅ Correct | ❌ Wrong |
|-----------|----------|
| `PROVIDER_CLIENT_ID` | `SPOTIFY_CLIENT_ID`, `LINEAR_CLIENT_ID` |
| `PROVIDER_CLIENT_SECRET` | `SPOTIFY_CLIENT_SECRET`, `GMAIL_SECRET` |
| `PROVIDER_ACCOUNTS_URL` | `SPOTIFY_ACCOUNTS_URL` |
| `PROVIDER_API_URL` | `LINEAR_API_URL`, `GITHUB_API_URL` |

**Why?**
- Same env var names work across all servers (Spotify, Linear, Gmail, etc.)
- Deployment scripts don't need service-specific logic
- `.env.example` and `wrangler.toml` remain generic templates
- Easier to audit security (one pattern to check)

**Example `.env`:**
```env
# Generic provider config — same vars for any OAuth provider
PROVIDER_CLIENT_ID=your-client-id
PROVIDER_CLIENT_SECRET=your-client-secret
PROVIDER_ACCOUNTS_URL=https://accounts.spotify.com   # or github.com, etc.
PROVIDER_API_URL=https://api.spotify.com             # optional, for API calls
```

**Exception:** If a server integrates multiple providers simultaneously (rare), prefix with provider name: `GITHUB_CLIENT_ID`, `GITLAB_CLIENT_ID`. Single-provider servers should always use `PROVIDER_*`.

### Auth strategies

Five auth strategies, configured via `AUTH_STRATEGY` env var:

| Strategy | Header | Use Case |
|----------|--------|----------|
| `oauth` | `Authorization: Bearer <RS_TOKEN>` | Full OAuth 2.1 PKCE flow with RS token → provider token mapping |
| `bearer` | `Authorization: Bearer <TOKEN>` | Static token from `BEARER_TOKEN` env |
| `api_key` | `X-Api-Key: <KEY>` (configurable) | Static key from `API_KEY` env |
| `custom` | Multiple headers | Custom headers from `CUSTOM_HEADERS` env |
| `none` | — | No authentication |

**OAuth flow (strategy=oauth):**
1. Client discovers AS metadata via `/.well-known/oauth-authorization-server`
2. Client initiates PKCE flow → `/authorize` → provider login
3. Provider callback → server issues RS tokens (access + refresh)
4. Client sends RS token → server maps to provider token → tool executes with provider API

**Token storage** (RS token → provider token mapping):
- `FileTokenStore` — Node.js, file-based with optional encryption
- `MemoryTokenStore` — Both runtimes, in-memory with TTL
- `KvTokenStore` — Workers, Cloudflare KV with optional encryption
- All support AES-256-GCM encryption via `RS_TOKENS_ENC_KEY`

## Sessions

Sessions enable multi-tenant operation. One server instance can serve multiple users with isolated state. Both runtimes use `SessionStore` for this.

**What sessions give you:**
- API key → session binding (who owns this connection)
- Session limits per API key (default: 5, LRU eviction)
- Session validation on every request (404 for invalid/expired sessions)
- Protocol version tracking per session
- Server→client request routing (sampling/elicitation need to know which client)

**What sessions don't give you (that's on the agent):**
- Conversation memory ("reply to that email")
- Workflow state (draft continuation, last issue ID)
- Context carryover between tool calls

**Storage implementations:**

| Store | Runtime | Backend | Persistence |
|-------|---------|---------|-------------|
| `MemorySessionStore` | Both | In-memory Map | Process lifetime |
| `SqliteSessionStore` | Node.js | SQLite via Drizzle | Disk |
| `KvSessionStore` | Workers | Cloudflare KV | Global |

**Session lifecycle (per MCP spec):**
1. Client sends `initialize` request without `Mcp-Session-Id` header
2. Server creates session via `SessionStore.create(sessionId, apiKey)`, returns session ID in response header
3. Client sends `initialized` notification with `Mcp-Session-Id` → server marks session as initialized
4. All subsequent requests must include `Mcp-Session-Id` (400 Bad Request if missing)
5. Server validates session exists on every request (404 Not Found if invalid/expired)
6. Session expires after TTL (default: 24h) or client sends DELETE request

**API key resolution** (for session binding):
- `X-Api-Key` or `X-Auth-Token` header (direct API key auth)
- Bearer token from `Authorization` header (OAuth RS token)
- Static `API_KEY` from config (fallback)
- `"public"` (unauthenticated)

**Multi-tenant model:**
```
User A (api_key_1) ──┐
                     │
User B (api_key_2) ──┼──▶ Single MCP Server ──▶ Provider API
                     │    (sessions isolate users)
User C (api_key_3) ──┘
```

## Adding tools

**Location:** `src/shared/tools/`

**Pattern:** schema → metadata → handler → register

```typescript
// 1. Define input schema with Zod
export const myToolInputSchema = z.object({
  query: z.string().describe('Search query'),
});

// 2. Create tool with defineTool()
export const myTool = defineTool({
  name: 'my_tool',
  title: 'My Tool',
  description: 'What it does',
  inputSchema: myToolInputSchema,
  outputSchema: { result: z.string() },  // optional
  annotations: {
    readOnlyHint: true,
    destructiveHint: false,
  },
  handler: async (args, context) => {
    // 3. Implement handler
    return {
      content: [{ type: 'text', text: args.query }],
      structuredContent: { result: args.query },  // required if outputSchema defined
    };
  },
});

// 4. Add to sharedTools array in registry.ts
export const sharedTools: RegisteredTool[] = [
  asRegisteredTool(healthTool),
  asRegisteredTool(echoTool),
  asRegisteredTool(myTool),  // ← add your tool here
];
```

**Annotations** control how clients display/invoke: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`.

**Services:** For complex integrations, put business logic in `src/shared/services/`. Extract when: handler exceeds ~30 lines, multiple tools share logic, or external API needs rate limiting/retries. Simple tools can keep logic inline.

## Known limitations

**Node.js runtime** — Full MCP support including server→client requests (sampling, elicitation, roots) via SDK's `StreamableHTTPServerTransport`. Sessions persist via `MemorySessionStore` (default) or `SqliteSessionStore` for disk persistence.

**Cloudflare Workers runtime** — Request→response mode only. Sessions persist via `KvSessionStore` across requests, but transport state is stateless (no SSE streams). Server→client requests (sampling, elicitation, roots) aren't available because they require an active SSE stream which Workers can't maintain. Use Workers for simple tool servers; for full MCP features, use Node.js or implement Durable Objects.

## License

MIT
