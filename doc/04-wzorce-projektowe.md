# 04 — Wzorce projektowe w kodzie

Ten rozdział mapuje konkretne pliki w projekcie na klasyczne wzorce projektowe. Rozumiejąc *dlaczego* użyto danego wzorca, łatwiej rozumieć *co* kod robi.

---

## 4.1 Strategy Pattern — strategie autoryzacji

**Problem:** Mamy pięć sposobów autoryzacji (OAuth, Bearer, API Key, Custom, None). Kod narzędzi nie powinien wiedzieć, która strategia jest aktywna.

**Wzorzec Strategy** definiuje rodzinę algorytmów (strategie), enkapsuluje każdy z nich i sprawia, że są wymienne.

```typescript
// src/shared/auth/strategy.ts

// "Interfejs" strategii (TypeScript union type)
type AuthStrategy = 'oauth' | 'bearer' | 'api_key' | 'custom' | 'none';

// Każda strategia to funkcja o tym samym podpisie:
//   (config) → { headers: Record<string, string> }

// Strategia OAuth: tokeny z TokenStore, dynamiczne
// Strategia bearer: statyczny token z env
// Strategia api_key: klucz z env do konkretnego nagłówka
// Strategia custom: wiele nagłówków z env
// Strategia none: puste headers

function resolveStaticAuth(config: AuthConfig): ResolvedAuth {
  switch (config.strategy) {
    case 'bearer': return { headers: { Authorization: `Bearer ${config.bearerToken}` } };
    case 'api_key': return { headers: { [config.apiKeyHeader]: config.apiKey } };
    case 'custom':  return { headers: parseCustomHeaders(config.customHeaders) };
    case 'none':    return { headers: {} };
    // 'oauth' jest obsługiwany osobno (wymaga async TokenStore lookup)
  }
}
```

**Wynik:** Tool handler zawsze dostaje `context.providerToken` lub `context.resolvedHeaders` — bez warunku `if (strategy === 'oauth')`.

**Gdzie:** `src/shared/auth/strategy.ts`, `src/shared/types/auth.ts`

---

## 4.2 Registry Pattern — rejestr narzędzi

**Problem:** Serwer musi dynamicznie znajdować i wywoływać narzędzia po nazwie (`tools/call { name: "search_emails" }`).

**Wzorzec Registry** to centralne repozytorium obiektów, z których można pobierać elementy po kluczu.

```typescript
// src/shared/tools/registry.ts

// Rejestr: tablica wszystkich narzędzi
export const sharedTools: RegisteredTool[] = [
  asRegisteredTool(healthTool),
  asRegisteredTool(echoTool),
  // Twoje narzędzia tu:
  asRegisteredTool(myTool),
];

// Lookup po nazwie:
export function getSharedTool(name: string): RegisteredTool | undefined {
  return sharedTools.find(t => t.name === name);
}

// Wykonanie z walidacją:
export async function executeSharedTool(
  name: string,
  args: unknown,
  context: ToolContext,
): Promise<ToolResult> {
  const tool = getSharedTool(name);
  if (!tool) throw new McpError(ErrorCode.MethodNotFound, `Unknown tool: ${name}`);

  // Walidacja input przez Zod
  const parsed = tool.inputSchema.safeParse(args);
  if (!parsed.success) throw new McpError(ErrorCode.InvalidParams, ...);

  return tool.handler(parsed.data, context);
}
```

**Dlaczego tablica zamiast Map?** Tablica zachowuje kolejność (listowanie narzędzi jest deterministyczne), Map jest szybsza dla lookup ale tu rozmiar jest mały (dziesiątki, nie tysiące).

**Gdzie:** `src/shared/tools/registry.ts`

---

## 4.3 Factory Pattern — budowanie serwera MCP

**Problem:** Serwer MCP wymaga konfiguracji (rejestracja tools, prompts, resources). Proces budowania jest złożony — wydzielenie go do osobnej funkcji czyści entry points.

**Wzorzec Factory** enkapsuluje logikę tworzenia obiektu.

```typescript
// src/core/mcp.ts

export function buildServer(options: ServerOptions): McpServer {
  const server = new McpServer({
    name: config.title,
    version: config.version,
  });

  // 1. Zarejestruj narzędzia
  registerTools(server, options.contextResolver);

  // 2. Zarejestruj prompts (jeśli zdefiniowane)
  if (prompts.length > 0) {
    registerPrompts(server);
  }

  // 3. Zarejestruj resources (jeśli zdefiniowane)
  if (resources.length > 0) {
    registerResources(server);
  }

  // 4. Opcjonalny callback po inicjalizacji
  if (options.onInitialized) {
    server.server.setRequestHandler(InitializedNotificationSchema, options.onInitialized);
  }

  return server;
}
```

Entry point (index.ts) wywołuje po prostu `buildServer(options)` — nie wie nic o kolejności rejestracji.

**Gdzie:** `src/core/mcp.ts`

---

## 4.4 Singleton Pattern — inicjalizacja storage

**Problem:** `TokenStore` i `SessionStore` powinny być zainicjalizowane raz (przy starcie aplikacji) i dostępne globalnie w całym procesie. Wielokrotna inicjalizacja = wielokrotne połączenia do pliku/bazy.

**Wzorzec Singleton** zapewnia, że klasa ma tylko jedną instancję.

```typescript
// src/shared/storage/singleton.ts

let tokenStore: TokenStore | null = null;
let sessionStore: SessionStore | null = null;

// Wywoływane raz przy starcie:
export function initializeStorage(ts: TokenStore, ss: SessionStore): void {
  if (tokenStore !== null) {
    throw new Error('Storage already initialized'); // guard
  }
  tokenStore = ts;
  sessionStore = ss;
}

// Wywoływane wszędzie gdzie potrzebne:
export function getTokenStore(): TokenStore {
  if (!tokenStore) throw new Error('Storage not initialized');
  return tokenStore;
}
```

**Uwaga na Workers:** W Workers `initializeStorage` jest wywoływane przy każdym requeście (bo Worker jest "nowy" dla każdego requestu). Guard `if (tokenStore !== null)` jest potrzebny żeby nie nadpisać instancji przy wielokrotnym wywołaniu w tym samym procesie Workers (w praktyce Workers może reużywać globalny stan między requestami w tej samej instancji).

**Gdzie:** `src/shared/storage/singleton.ts`

---

## 4.5 AsyncLocalStorage — kontekst bez prop-drillingu

To jest jeden z najbardziej eleganckich wzorców w tym kodzie i warto go dobrze zrozumieć.

**Problem:** Handler narzędzia potrzebuje informacji o bieżącym requeście (kto wywołał, jaki token auth, jaki sessionId). Jak przekazać te dane bez przekazywania ich przez każdą warstwę kodu?

**Naiwne rozwiązanie (prop-drilling):**
```typescript
// Każda funkcja musi przyjmować i przekazywać context
function handleRequest(req, context) {
  processAuth(req, context);
}
function processAuth(req, context) {
  executeTools(req, context);  // context przekazywany w kółko
}
function executeTools(req, context) {
  runTool(req, context);       // nawet jeśli nie używa
}
```

**AsyncLocalStorage** — Node.js odpowiednik na "thread-local storage":

```typescript
// src/core/context.ts

const AuthContextStorage = new AsyncLocalStorage<RequestContext>();

// Na początku każdego requestu:
AuthContextStorage.run(requestContext, () => {
  // Cały async call-graph w tym bloku ma dostęp do context
  handleMcpRequest(request);
});

// Gdziekolwiek w call-graph (tools, oauth, logger):
function getCurrentAuthContext(): RequestContext | undefined {
  return AuthContextStorage.getStore();
}
```

**Jak to działa pod spodem?** `AsyncLocalStorage` śledzi aktywny kontekst przez granicę asynchroniczną — `await`, `setTimeout`, `Promise.then`. Każde `async` wywołanie "dziedziczy" kontekst rodzica. To jest specyfika V8/Node.js — każdy mikrotask ma powiązaną "execution context chain".

**Backup: contextRegistry Map:**
```typescript
// Równolegle z AsyncLocalStorage, kontekst jest też w globalnej mapie
const contextRegistry = new Map<string | number, RequestContext>();

// Po zakończeniu requestu:
contextRegistry.delete(requestId);
```

Dlaczego dwie metody? `AsyncLocalStorage` jest czyste i automatyczne, ale ma ograniczenia (nie działa przez Worker threads). Map jest fallbackiem i pozwala debugować aktywne konteksty.

**Gdzie:** `src/core/context.ts`

---

## 4.6 Repository Pattern — warstwy danych

**Problem:** Kod logiki biznesowej nie powinien znać szczegółów przechowywania danych (plik JSON? SQLite? KV w chmurze?).

**Wzorzec Repository** definiuje interfejs dostępu do danych. Implementacja może być dowolna.

```typescript
// "Interfejs" (src/shared/storage/)
interface TokenStore {
  get(key: string): Promise<TokenEntry | null>;
  set(key: string, value: TokenEntry, ttlSeconds?: number): Promise<void>;
  delete(key: string): Promise<void>;
  cleanup?(): Promise<void>;
}

// Implementacja A: plik JSON (Node.js)
class FileTokenStore implements TokenStore {
  private data = new Map<string, TokenEntry>();
  async get(key) { return this.data.get(key) ?? null; }
  // ...zapisuje do pliku co N sekund
}

// Implementacja B: Cloudflare KV (Workers)
class KvTokenStore implements TokenStore {
  constructor(private kv: KVNamespace) {}
  async get(key) { return this.kv.get(key, 'json'); }
  // ...TTL obsługiwany przez KV natively
}
```

OAuth flow, tool handlers, middleware — wszystkie operują na `TokenStore` interface, nigdy na konkretnej klasie.

---

## 4.7 Builder Pattern — `defineTool()`

**Problem:** Tworzenie narzędzia wymaga wielu pól, walidacji typów, type-erasure dla rejestru. Jak uczynić to wygodnym i bezpiecznym?

```typescript
// src/shared/tools/types.ts

// defineTool() to typed builder z generic'ami Zod
export function defineTool<T extends ZodRawShape>(
  definition: SharedToolDefinition<T>
): SharedToolDefinition<T> {
  return definition; // runtime: identity function
  // compile-time: pełna inferencja typów z Zod schema
}

// Użycie:
const myTool = defineTool({
  name: 'my_tool',
  inputSchema: z.object({
    query: z.string(),  // ← TypeScript wie że args.query to string
  }),
  handler: async (args, context) => {
    // args jest AUTOMATYCZNIE typowany jako { query: string }
    // bez żadnych as-castów
    console.log(args.query.toUpperCase()); // ✓ TypeScript OK
  },
});
```

`asRegisteredTool()` to osobna funkcja, która wykonuje type-erasure (zamienia generic type na `unknown`) żeby móc umieścić narzędzia różnych typów w jednej tablicy `sharedTools[]`.

---

## 4.8 Middleware Chain — bezpieczeństwo requestu

**Problem:** Każdy request do `/mcp` musi przejść przez sekwencję kroków: CORS → auth → session validation → rate limit → handler.

**Middleware chain** to sekwencja funkcji, z których każda może zmodyfikować request/response lub przerwać przetwarzanie.

```typescript
// src/adapters/http-hono/middleware.security.ts (uproszczony)

// Hono middleware — każda funkcja woła next() lub zwraca Response
app.use('/mcp', async (c, next) => {
  // 1. Sprawdź CORS
  if (!isAllowedOrigin(c.req.header('origin'))) {
    return c.json({ error: 'Forbidden' }, 403);
  }
  await next();
});

app.use('/mcp', async (c, next) => {
  // 2. Wyciągnij i zwaliduj auth
  const token = extractBearerToken(c.req.header('authorization'));
  if (!token && authRequired) {
    return c.json({ error: 'Unauthorized' }, 401, {
      'WWW-Authenticate': 'Bearer realm="mcp"'
    });
  }
  c.set('token', token);
  await next();
});

app.post('/mcp', async (c) => {
  // 3. Właściwy handler — token już jest w kontekście
  const token = c.get('token');
  // ...
});
```

**Fail-fast principle:** Każdy middleware weryfikuje jeden warunek. Jeśli warunek nie jest spełniony, request jest odrzucany natychmiast bez dalszego przetwarzania. Oszczędza zasoby i upraszcza logikę.

---

## Podsumowanie wzorców

| Wzorzec | Plik | Problem jaki rozwiązuje |
|---------|------|------------------------|
| Strategy | `src/shared/auth/strategy.ts` | Wymienne strategie auth bez if/else w handlerach |
| Registry | `src/shared/tools/registry.ts` | Dynamiczne wyszukiwanie narzędzi po nazwie |
| Factory | `src/core/mcp.ts` | Enkapsulacja złożonego tworzenia serwera MCP |
| Singleton | `src/shared/storage/singleton.ts` | Jedna instancja TokenStore/SessionStore |
| AsyncLocalStorage | `src/core/context.ts` | Kontekst requestu bez prop-drillingu |
| Repository | `src/shared/storage/` | Niezależność logiki od implementacji storage |
| Builder + Type Inference | `src/shared/tools/types.ts` | Bezpieczne typowanie z Zod bez boilerplatu |
| Middleware Chain | `src/adapters/http-hono/` | Sekwencja weryfikacji requestu z fail-fast |

**Dalej:** [05 — Warstwa przechowywania danych](./05-warstwa-storage.md)
