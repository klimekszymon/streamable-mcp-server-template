# 02 — Architektura dualna: Node.js i Cloudflare Workers

## 2.1 Cel: jeden kod, dwa środowiska

Ten szablon kompiluje się do dwóch niezależnych binariów z jednej bazy kodu TypeScript. To nieoczywiste architektonicznie — jak to osiągnąć bez kopiowania logiki?

```
src/
├── index.ts          ← entry point Node.js
├── worker.ts         ← entry point Cloudflare Workers
├── shared/           ← współdzielona logika (ok. 90% kodu)
│   ├── tools/
│   ├── oauth/
│   ├── storage/
│   └── ...
├── adapters/
│   ├── http-hono/    ← adapter specyficzny dla Node.js/Hono
│   └── http-workers/ ← adapter specyficzny dla Workers/itty-router
└── http/             ← Node.js-only (OAuth Authorization Server)
```

**Zasada:** Logika biznesowa żyje w `shared/`. Specyfika środowiska żyje w `adapters/`.

---

## 2.2 Wzorzec Adaptera

To jeden z klasycznych wzorców projektowych (Gang of Four). Pozwala dwóm niekompatybilnym interfejsom współpracować przez warstwę pośrednią.

```
Problem: Hono i itty-router mają różne API
         MCP SDK i własny dispatcher mają różny kod transportu
         FileTokenStore i KvTokenStore mają różny backend

Rozwiązanie:
  Wspólny interfejs ──┬── Implementacja A (Hono)
                      └── Implementacja B (itty-router)
```

Przykład z kodu — interfejs `TokenStore`:

```typescript
// src/shared/storage/ (uproszczony)
interface TokenStore {
  get(rsToken: string): Promise<TokenData | null>;
  set(rsToken: string, data: TokenData, ttl?: number): Promise<void>;
  delete(rsToken: string): Promise<void>;
}

// Implementacja A: Node.js, plik JSON
class FileTokenStore implements TokenStore { ... }

// Implementacja B: Cloudflare Workers, KV
class KvTokenStore implements TokenStore { ... }

// Implementacja C: oba środowiska, pamięć (testy/dev)
class MemoryTokenStore implements TokenStore { ... }
```

Kod narzędzi i OAuth nie wie, *gdzie* są przechowywane tokeny. Zna tylko interfejs `TokenStore`.

---

## 2.3 Różnice między środowiskami

| Aspekt | Node.js | Cloudflare Workers |
|--------|---------|-------------------|
| Model wykonania | Długo żyjący proces | Funkcja na żądanie |
| Pamięć między requestami | Tak (zmienne globalne) | Nie (każdy request to "nowy" worker) |
| Czas wykonania | Nieograniczony | Maks. ~30s (CPU) |
| SSE / WebSocket | Tak | Ograniczone |
| System plików | Tak | Nie |
| Biblioteki npm | Wszystkie | Podzbiór (Web APIs + compat) |

### Konsekwencja dla sesji

W Node.js sesje mogą żyć w `Map` w pamięci — proces żyje długo. W Workers każdy request to potencjalnie inny worker (inna instancja), więc sesje *muszą* być zewnętrzne (Cloudflare KV).

```typescript
// Node.js: src/index.ts
const sessionStore = new MemorySessionStore();  // pamięć wystarczy

// Workers: src/worker.ts
const sessionStore = new KvSessionStore(env.TOKENS);  // KV bo stateless
```

### Konsekwencja dla transportu MCP

MCP SDK ma `StreamableHTTPServerTransport` — wspiera SSE, bidirectional requests. Ale wymaga Node.js APIs (streams, EventEmitter itp.).

W Workers zamiast SDK transport, jest własny dispatcher:

```typescript
// src/shared/mcp/dispatcher.ts — uproszczony schemat
async function dispatch(request: JsonRpcRequest, context: ToolContext) {
  switch (request.method) {
    case 'tools/list':    return listTools();
    case 'tools/call':    return executeTool(request.params, context);
    case 'resources/read': return readResource(request.params);
    // ...
  }
}
```

To jest "ręczna" reimplementacja części SDK. Ograniczenie: brak SSE = brak sampling, elicitation, roots.

---

## 2.4 Entry points — co się dzieje przy starcie

### Node.js (`src/index.ts`)

```
1. initializeStorage(FileTokenStore, MemorySessionStore)
   └── singleton — jeden store na cały proces

2. createHonoApp(storage)
   └── MCP routes na /mcp
   └── Health check na /health
   └── Discovery endpoints

3. Jeśli AUTH_ENABLED=true:
   └── createAuthApp() na PORT+1
   └── Oddzielny serwer HTTP tylko dla OAuth

4. Graceful shutdown handlers
   └── SIGTERM/SIGINT → flush + cleanup
```

**Dlaczego OAuth na osobnym porcie?** Klient MCP komunikuje się z `/mcp`. OAuth Authorization Server (`/authorize`, `/token`, `/.well-known/*`) to oddzielna rola. Separacja portów ułatwia konfigurację reverse proxy — możesz wystawić obie usługi pod różnymi domenami lub ścieżkami.

### Cloudflare Workers (`src/worker.ts`)

```
1. Shim process.env z env bindings
   └── Workers używa env jako obiekt, Node.js używa process.env

2. initializeStorage(KvTokenStore, KvSessionStore)
   └── wywoływane na KAŻDY request (ale singleton guard!)

3. router.fetch(request, env)
   └── itty-router dopasowuje ścieżkę
   └── zwraca Response
```

**Co to jest "shim"?** Kod, który symuluje API jednego środowiska w innym. Workers nie ma `process.env` — jest `env` przekazywany jako argument. Shim kopiuje wartości do `process.env` żeby reszta kodu działała identycznie.

---

## 2.5 Hono vs itty-router

Oba to frameworki HTTP, ale o różnych priorytetach:

**Hono** — pełnofeatured, Koa-inspired, middleware chain:
```typescript
const app = new Hono();
app.use('*', corsMiddleware());      // middleware globalne
app.post('/mcp', authMiddleware(), mcpHandler);
```

**itty-router** — minimalistyczny, <1KB, edge-first:
```typescript
const router = Router();
router.post('/mcp', withCors, withAuth, handleMcp);
```

W Workers minimalizm ma znaczenie — mniejszy bundle = szybszy cold start (pierwsze uruchomienie workera po pauzie). Hono też działa na Workers, ale itty-router jest prostszy przy ograniczonym zestawie funkcji.

---

## 2.6 Gdzie pisać kod i gdzie nie

Reguła dla współtwórców:

| Gdzie | Co tam trafia |
|-------|--------------|
| `src/shared/tools/` | Twoje narzędzia — *zawsze tu* |
| `src/shared/oauth/` | Logika OAuth — *nie modyfikuj bez potrzeby* |
| `src/shared/storage/` | Nowe implementacje store — *dodawaj nowe, nie edytuj interfejsu* |
| `src/adapters/http-hono/` | Middleware/routes specyficzne dla Hono |
| `src/adapters/http-workers/` | Handlery specyficzne dla Workers |
| `src/http/` | Serwer OAuth (Node.js only) |
| `src/config/metadata.ts` | Metadane twoich narzędzi |

**Zasada prosta:** jeśli piszesz kod narzędzia i czujesz że musisz edytować plik w `adapters/` — prawdopodobnie robisz coś nie tak. Narzędzia nie wiedzą o adapterach.

---

## 2.7 Schemat przepływu requestu

Pełny przepływ żądania `tools/call` w Node.js:

```
HTTP POST /mcp
│
└── Hono router (src/http/routes/mcp.ts)
    │
    └── Auth middleware (src/adapters/http-hono/middleware.security.ts)
        │   Wyciąga token z nagłówka Authorization
        │   Mapuje RS token → provider token (TokenStore)
        │   Tworzy RequestContext w AsyncLocalStorage
        │
        └── MCP SDK StreamableHTTPServerTransport
            │   Parsuje JSON-RPC z body requestu
            │
            └── McpServer (src/core/mcp.ts)
                │   Dopasowuje metodę (tools/call)
                │
                └── executeSharedTool() (src/shared/tools/registry.ts)
                    │   Waliduje input Zod
                    │   Wywołuje handler
                    │
                    └── Twój handler (src/shared/tools/my-tool.ts)
                            Używa context.providerToken do API call
                            Zwraca ToolResult
```

---

## Podsumowanie

| Wzorzec | Zastosowanie |
|---------|-------------|
| Adapter | Jeden interfejs, wiele implementacji (TokenStore, SessionStore) |
| Shared code | Logika biznesowa niezależna od środowiska (shared/) |
| Shim | Symulacja process.env na Workers |
| Graceful shutdown | Bezpieczne zakończenie Node.js z flushem danych |

**Dalej:** [03 — OAuth 2.1 i wzorzec podwójnego tokenu](./03-oauth-i-podwojny-token.md)
