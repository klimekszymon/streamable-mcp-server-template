# 06 — Jak dodać własne narzędzie (tool)

Praktyczny przewodnik od zera do działającego narzędzia. Przykład: narzędzie pobierające listę repozytoriów z GitHub API.

---

## 6.1 Przepływ tworzenia narzędzia

```
1. Zdefiniuj schemat wejściowy (Zod)
2. Napisz serwis (klient HTTP do zewnętrznego API)
3. Utwórz handler narzędzia (defineTool)
4. Zarejestruj w registry
5. Przetestuj
```

---

## 6.2 Krok 1 — Schemat wejściowy Zod

Schemat jest jednocześnie:
- walidacją input (runtime)
- typowaniem dla TypeScript (compile-time)
- dokumentacją dla modelu AI (`.describe()`)

```typescript
// src/shared/tools/github/list-repos.ts
import { z } from 'zod';

const ListReposInputSchema = z.object({
  username: z
    .string()
    .min(1)
    .describe('GitHub username, e.g. "torvalds"'),

  type: z
    .enum(['all', 'owner', 'member'])
    .optional()
    .default('owner')
    .describe('Filter repos by type. Default: "owner".'),

  limit: z
    .number()
    .int()
    .min(1)
    .max(100)
    .optional()
    .default(30)
    .describe('Max repos to return (1-100). Default: 30.'),
});
```

**Zasady dobrych opisów (`.describe()`):**
- Pisz do modelu AI, nie do programisty
- Podaj przykłady wartości
- Dokumentuj wartości domyślne
- Opisuj ograniczenia (min/max)

---

## 6.3 Krok 2 — Serwis (klient API)

Serwis to prosta klasa lub funkcja hermetyzująca wywołania HTTP. Nie wie nic o MCP.

```typescript
// src/shared/services/github-client.ts
const GITHUB_API = 'https://api.github.com';

export interface GitHubRepo {
  id: number;
  name: string;
  full_name: string;
  description: string | null;
  html_url: string;
  stargazers_count: number;
  language: string | null;
  private: boolean;
}

export class GitHubClient {
  constructor(private accessToken: string) {}

  private async request<T>(path: string): Promise<T> {
    const response = await fetch(`${GITHUB_API}${path}`, {
      headers: {
        Authorization: `Bearer ${this.accessToken}`,
        Accept: 'application/vnd.github.v3+json',
        'User-Agent': 'MCP-Server/1.0',
      },
    });

    if (response.status === 401) {
      throw new Error('GitHub token expired or invalid. Please re-authenticate.');
    }

    if (response.status === 404) {
      throw new Error(`Not found: ${path}`);
    }

    if (!response.ok) {
      throw new Error(`GitHub API error: ${response.status}`);
    }

    return response.json() as Promise<T>;
  }

  async listRepos(username: string, type: string, perPage: number): Promise<GitHubRepo[]> {
    return this.request(`/users/${username}/repos?type=${type}&per_page=${perPage}`);
  }
}
```

**Dlaczego osobna klasa?** Gdy masz 10 narzędzi GitHub, wszystkie reużywają jednego klienta. Zmiana base URL, nagłówków czy error handling → jedna zmiana w jednym miejscu.

---

## 6.4 Krok 3 — Handler narzędzia

```typescript
// src/shared/tools/github/list-repos.ts (pełny plik)
import { z } from 'zod';
import { defineTool, type ToolContext, type ToolResult } from '../types.js';
import { GitHubClient } from '../../services/github-client.js';

const ListReposInputSchema = z.object({
  username: z.string().min(1).describe('GitHub username'),
  type: z.enum(['all', 'owner', 'member']).optional().default('owner'),
  limit: z.number().int().min(1).max(100).optional().default(30),
});

export const listReposTool = defineTool({
  name: 'list_repos',
  title: 'List GitHub Repositories',
  description: `List public repositories for a GitHub user.
Returns name, description, stars, language, and URL.
Use when the user asks about someone's GitHub projects or repos.`,

  inputSchema: ListReposInputSchema,

  annotations: {
    readOnlyHint: true,      // nie modyfikuje danych
    destructiveHint: false,
    openWorldHint: true,     // sięga do zewnętrznego API
  },

  handler: async (args, context: ToolContext): Promise<ToolResult> => {
    // 1. Pobierz token z context
    const token = context.providerToken ?? context.provider?.accessToken;
    if (!token) {
      return {
        isError: true,
        content: [{ type: 'text', text: 'Authentication required. Please sign in with GitHub.' }],
      };
    }

    // 2. Sprawdź anulowanie (ważne dla długich operacji)
    if (context.signal?.aborted) {
      return {
        isError: true,
        content: [{ type: 'text', text: 'Operation cancelled.' }],
      };
    }

    try {
      // 3. Wywołaj API
      const client = new GitHubClient(token);
      const repos = await client.listRepos(args.username, args.type, args.limit);

      // 4. Formatuj dla modelu AI (zwięźle, czytelnie)
      if (repos.length === 0) {
        return {
          content: [{ type: 'text', text: `No repositories found for ${args.username}.` }],
        };
      }

      const formatted = repos
        .map(r =>
          `**${r.name}** (⭐ ${r.stargazers_count})\n` +
          `${r.description ?? 'No description'}\n` +
          `Language: ${r.language ?? 'Unknown'} | ${r.html_url}`
        )
        .join('\n\n');

      return {
        content: [{
          type: 'text',
          text: `Found ${repos.length} repos for ${args.username}:\n\n${formatted}`,
        }],
        // Opcjonalnie: strukturalne dane dla klientów które je obsługują
        structuredContent: {
          username: args.username,
          count: repos.length,
          repos: repos.map(r => ({
            name: r.name,
            stars: r.stargazers_count,
            language: r.language,
            url: r.html_url,
          })),
        },
      };
    } catch (error) {
      // 5. Błędy API → czytelne komunikaty
      const message = error instanceof Error ? error.message : 'Unknown error';
      return {
        isError: true,
        content: [{ type: 'text', text: `Failed to list repos: ${message}` }],
      };
    }
  },
});
```

**Ważne zasady handlera:**

| Zasada | Powód |
|--------|-------|
| Sprawdź token na początku | Nie wykonuj żadnych operacji bez auth |
| Sprawdź `context.signal?.aborted` | Użytkownik mógł anulować request |
| Zwróć `isError: true` zamiast rzucać | MCP oczekuje structured errors, nie exceptiony |
| Formatuj text dla LLM | Model AI czyta text, nie JSON. Czytelna odpowiedź = lepsza kontynuacja konwersacji |
| Obsłuż 401 osobno | Błąd auth wymaga re-autoryzacji, nie jest "błędem API" |

---

## 6.5 Krok 4 — Rejestracja w registry

```typescript
// src/shared/tools/registry.ts
import { listReposTool } from './github/list-repos.js';
// Jeśli masz więcej narzędzi GitHub:
import { getRepoTool } from './github/get-repo.js';
import { listIssuestool } from './github/list-issues.js';

export const sharedTools: RegisteredTool[] = [
  // Usuń przykładowe narzędzia jeśli nie potrzebujesz:
  // asRegisteredTool(healthTool),
  // asRegisteredTool(echoTool),

  // Dodaj swoje:
  asRegisteredTool(listReposTool),
  asRegisteredTool(getRepoTool),
  asRegisteredTool(listIssuestool),
];
```

**Co robi `asRegisteredTool()`?** Wykonuje type-erasure: zamienia `SharedToolDefinition<{ username: string, ... }>` na `RegisteredTool` z `inputSchema: ZodObject<ZodRawShape>`. Bez tego nie możesz wrzucić narzędzi różnych typów do jednej tablicy.

---

## 6.6 Krok 5 — Testowanie

### Test ręczny przez curl

```bash
# Uruchom serwer (bez auth dla szybkiego testu)
AUTH_STRATEGY=none bun dev

# Sprawdź listę narzędzi
curl -s -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"1"}}}' | jq .

# (Zapamiętaj Mcp-Session-Id z odpowiedzi!)
SESSION_ID="..."

curl -s -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' | jq '.result.tools[].name'

# Wywołaj narzędzie
curl -s -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc":"2.0","id":3,
    "method":"tools/call",
    "params":{
      "name":"list_repos",
      "arguments":{"username":"torvalds","limit":5}
    }
  }' | jq '.result.content[0].text'
```

### MCP Inspector (UI)

```bash
bun run mcp:inspector
# Otwiera przeglądarkę z GUI do testowania narzędzi interaktywnie
```

---

## 6.7 ToolContext — co masz do dyspozycji w handlerze

```typescript
interface ToolContext {
  sessionId: string;            // ID sesji MCP klienta

  // Auth — zależy od AUTH_STRATEGY
  authStrategy?: AuthStrategy;  // 'oauth' | 'bearer' | 'api_key' | 'custom' | 'none'
  providerToken?: string;       // access token do zewnętrznego API (OAuth/bearer)
  provider?: {                  // pełne info OAuth
    accessToken: string;
    refreshToken?: string;
    expiresAt?: number;
    scopes?: string[];
  };
  resolvedHeaders?: Record<string, string>; // nagłówki dla strategii custom/api_key

  // Flow control
  signal?: AbortSignal;         // anulowanie requestu (sprawdzaj przy długich op.)

  // Metadata
  meta?: {
    progressToken?: string | number;  // token do wysyłania progress notifications
    requestId?: string | number;
  };
}
```

**Kiedy używać `provider` zamiast `providerToken`?**

Zazwyczaj wystarczy `providerToken` (to skrót do `provider.accessToken`). `provider` używasz gdy potrzebujesz sprawdzić scopes lub czas wygaśnięcia tokenu.

```typescript
// Sprawdzenie zakresu przed wywołaniem API
if (!context.provider?.scopes?.includes('repo:write')) {
  return {
    isError: true,
    content: [{ type: 'text', text: 'Missing required scope: repo:write. Re-authenticate with write access.' }],
  };
}
```

---

## 6.8 Progress notifications (opcjonalne)

Dla operacji trwających dłużej niż kilka sekund warto wysyłać informacje o postępie:

```typescript
// Sprawdź czy klient oczekuje progressu
const progressToken = context.meta?.progressToken;

// Wyślij progress (tylko Node.js z SDK)
if (progressToken) {
  await server.server.notification({
    method: 'notifications/progress',
    params: {
      progressToken,
      progress: 0.5,    // 0.0 - 1.0
      total: 100,
      message: 'Fetching page 5 of 10...',
    },
  });
}
```

**Uwaga:** To wymaga dostępu do instancji serwera MCP. Najprościej przekazać ją przez closure lub przez context. W Workers ten pattern nie jest dostępny (brak SSE).

---

## 6.9 Częste błędy i jak ich unikać

**❌ Rzucanie wyjątków z handlera**
```typescript
// Źle:
throw new Error('API failed');

// Dobrze:
return { isError: true, content: [{ type: 'text', text: 'API failed: ...' }] };
```
Wyjątki z handlera są łapane przez registry i konwertowane na MCP errors. Ale structured `isError: true` daje lepszą kontrolę nad formatem i treścią błędu.

**❌ Logowanie pełnych tokenów**
```typescript
// Źle:
console.log('Token:', context.providerToken);

// Dobrze:
console.log('Token present:', !!context.providerToken);
```

**❌ Ignorowanie anulowania**
```typescript
// Dla długich pętli:
for (const item of hugeList) {
  if (context.signal?.aborted) break; // ← sprawdzaj co iterację
  await processItem(item);
}
```

**❌ Zwracanie surowego JSON jako text**
```typescript
// Źle (model AI nie lubi parsować JSON):
return { content: [{ type: 'text', text: JSON.stringify(data) }] };

// Dobrze (czytelny tekst + structured data oddzielnie):
return {
  content: [{ type: 'text', text: `Found ${data.items.length} items: ${data.items.map(i => i.name).join(', ')}` }],
  structuredContent: data,
};
```

---

## 6.10 Checklista przed deployment

```
□ inputSchema ma .describe() na każdym polu
□ description narzędzia wyjaśnia KIEDY go używać
□ Handler sprawdza token na początku
□ Handler obsługuje context.signal?.aborted
□ Błędy API zwracają isError: true z user-friendly message
□ Narzędzie nie loguje tokenów ani PII
□ Dodano do sharedTools[] w registry.ts
□ Przetestowano przez MCP Inspector lub curl
□ Annotations są poprawne (readOnlyHint, destructiveHint)
```

---

## Podsumowanie przepływu

```
Definicja:
  z.object({...})          ← schemat + dokumentacja dla AI
  defineTool({...})        ← builder z type inference

Rejestracja:
  asRegisteredTool(tool)   ← type erasure
  sharedTools.push(...)    ← dodanie do rejestru

Wykonanie (automatyczne przez framework):
  tools/call request       ← JSON-RPC od klienta
  getSharedTool(name)      ← wyszukanie w registry
  schema.safeParse(args)   ← walidacja Zod
  handler(args, context)   ← twój kod
  ToolResult               ← odpowiedź do klienta
```

**Dokumentacja zakończona.** Wróć do [indeksu](./00-index.md) lub zacznij budować narzędzie.
