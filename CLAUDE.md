# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bun dev              # Node.js server (MCP :3000, OAuth AS :3001)
bun dev:worker       # Cloudflare Workers via wrangler
bun test             # Run all tests
bun run test:client  # Manual end-to-end test script
bun run mcp:inspector # Open MCP Inspector UI
bun run build        # Compile to dist/
bun run deploy       # Deploy to Cloudflare Workers
```

Linting/formatting uses **Biome** (not ESLint/Prettier):
```bash
bunx biome check src/     # Check for issues
bunx biome check --write src/  # Auto-fix
```

## Architecture Overview

### Dual-Runtime Design

The codebase targets two runtimes from shared code:

| Path | Runtime | Transport |
|------|---------|-----------|
| `src/index.ts` | Node.js | Hono + MCP SDK `StreamableHTTPServerTransport` |
| `src/worker.ts` | Cloudflare Workers | itty-router + custom JSON-RPC dispatcher |

The **Node.js path** gets full MCP SDK support including SSE-based features: sampling (server→client LLM requests), elicitation (asking user for input mid-tool), and roots (filesystem access hints).

The **Workers path** uses `src/shared/mcp/dispatcher.ts` — a hand-rolled JSON-RPC router — because the MCP SDK's transport layer requires Node.js APIs unavailable in Workers. Features are request/response only (no SSE, no server-initiated messages).

Runtime-specific code lives under `src/http/` (Node.js) and `src/adapters/` (both runtimes split into `http-hono/` and `http-workers/`). Everything else is under `src/shared/`.

### Auth Strategies

`src/shared/auth/strategy.ts` implements a union of five strategies: `oauth | bearer | api_key | custom | none`. All strategies funnel into the same `ToolContext.providerToken` field — tools don't know which auth method was used. Configured via `AUTH_STRATEGY` env var.

The OAuth path is the most complex: the server acts as an **OAuth Authorization Server** (`src/http/auth-app.ts`, port 3001) that fronts any upstream OAuth provider. It issues its own RS (Resource Server) tokens to MCP clients, then maps them to upstream provider tokens stored encrypted at rest. This double-token pattern means upstream credentials never leave the server.

### Request Context Flow

`src/core/context.ts` uses `AsyncLocalStorage` to thread `RequestContext` through async chains without prop-drilling. On each request a context is written to both AsyncLocalStorage and a `contextRegistry` Map (keyed by `requestId`). Tools access it via `getCurrentAuthContext()`.

### Tool System

All tools live in `src/shared/tools/`. Define a tool with:

```typescript
export const myTool = defineTool({
  name: 'my_tool',
  description: '...',
  inputSchema: z.object({ ... }),
  handler: async (args, context) => {
    // context.providerToken — upstream API token
    // context.signal — AbortSignal for cancellation
    return { content: [{ type: 'text', text: '...' }] };
  },
});
```

Register it by adding to the array in `src/shared/tools/registry.ts`. The registry handles Zod validation, error wrapping, and MCP content formatting for both runtimes.

### Storage Layer

Three token store implementations share one interface (`TokenStore`):

- `FileTokenStore` — Node.js, JSON file, optional AES-256-GCM encryption
- `MemoryTokenStore` — both runtimes, dev/testing
- `KvTokenStore` — Cloudflare Workers KV

Session stores follow the same pattern. Initialized once at startup via `src/shared/storage/singleton.ts` and accessed globally.

### Configuration

`src/shared/config/env.ts` is the single source of truth. It parses and validates all environment variables into a typed `UnifiedConfig` object. Copy `.env.example` to `.env` for local dev; for Workers, use `wrangler.example.toml` as a starting point.

## Key Patterns to Understand

- **`defineTool` + `asRegisteredTool`**: type-safe tool definition with Zod, then type-erased for the registry — see `src/shared/tools/registry.ts`
- **PKCE OAuth flow**: all OAuth requires S256 challenge — see `src/shared/oauth/flow.ts`
- **SSRF protection for CIMD**: SEP-991 client metadata documents are fetched with domain allowlisting — see `src/shared/oauth/ssrf.ts`
- **Proactive token refresh**: refresh happens before expiry, not on 401 — see `src/shared/oauth/refresh.ts`
- **`manual.md`**: step-by-step guide (9 phases) for converting this template into a real production MCP server — read this before starting implementation
