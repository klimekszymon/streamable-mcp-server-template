# 01 — Co to jest MCP i dlaczego potrzebujemy serwera?

## 1.1 Problem, który MCP rozwiązuje

Wyobraź sobie, że budujesz aplikację z asystentem AI. Chcesz, żeby mógł:
- Czytać maile (Gmail API)
- Tworzyć zadania (Linear API)
- Wyszukiwać kod (GitHub API)

Bez MCP musisz **dla każdej kombinacji** (model AI × serwis zewnętrzny) napisać osobny kod. Trzy modele AI × trzy serwisy = dziewięć integracji. Sześćdziesiąt modeli × sto serwisów = sześć tysięcy integracji.

To jest **problem N×M**.

```
Bez MCP:
  GPT-4    ──→ Gmail adapter
  GPT-4    ──→ Linear adapter
  Claude   ──→ Gmail adapter    (duplikacja!)
  Claude   ──→ Linear adapter   (duplikacja!)
  ...

Z MCP:
  GPT-4    ─┐
  Claude   ─┼──→ MCP Client ──→ MCP Server ──→ Gmail API
  Gemini   ─┘                ──→ MCP Server ──→ Linear API
```

MCP definiuje **wspólny protokół**: klient (model AI, IDE, aplikacja) wie jak rozmawiać z serwerem. Serwer wie jak obsłużyć żądania klienta. Nikt nie wie nic o szczegółach implementacji drugiej strony. To jest **separacja odpowiedzialności** na poziomie protokołu.

---

## 1.2 MCP to JSON-RPC 2.0

MCP nie wymyśla własnego formatu wiadomości. Korzysta z istniejącego standardu: **JSON-RPC 2.0**.

JSON-RPC 2.0 to prosty protokół wywołań zdalnych:

```json
// Żądanie (klient → serwer)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_emails",
    "arguments": { "query": "is:unread" }
  }
}

// Odpowiedź (serwer → klient)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "Znaleziono 5 nieprzeczytanych maili..." }
    ]
  }
}
```

**Dlaczego JSON-RPC, a nie REST?**

REST jest zorientowany na zasoby (`GET /emails/123`). JSON-RPC jest zorientowany na *akcje* (`call search_emails`). W przypadku narzędzi AI akcje są naturalniejszym modelem — model nie pobiera zasobu, on *wywołuje funkcję*.

---

## 1.3 Trzy rodzaje capabilities (możliwości serwera)

MCP definiuje trzy typy rzeczy, które serwer może udostępniać:

### Tools (narzędzia) — akcje

Narzędzia to funkcje z efektami ubocznymi lub bez. Model AI *decyduje* kiedy je wywołać.

```typescript
// Definicja po stronie serwera
{
  name: "send_email",
  description: "Wysyła email do odbiorcy",
  inputSchema: z.object({
    to: z.string().email(),
    subject: z.string(),
    body: z.string(),
  }),
}
// → model widzi opis i schemat, sam decyduje kiedy wywołać
```

**Kluczowa zasada:** Opis narzędzia jest *dokumentacją dla modelu AI*, nie dla programisty. Pisz go jak komunikat: "Use this tool when...", "Returns...", "Requires...".

### Resources (zasoby) — dane

Zasoby to dane tylko do odczytu identyfikowane przez URI. Klient może je subskrybować i otrzymywać powiadomienia o zmianach.

```typescript
// URI zasobu
"gmail://inbox/unread-count"     // liczba nieprzeczytanych
"github://repo/issues/open"       // lista otwartych issues
"config://server/capabilities"    // konfiguracja serwera
```

**Kiedy resource zamiast tool?** Gdy dane są statyczne lub zmieniają się rzadko i klient chce je *obserwować* (nie wywoływać akcji). Dla danych, które wymagają obliczeń lub wywołań API na żądanie — użyj tool.

### Prompts (szablony promptów) — szablony

Prompts to szablony wiadomości z parametrami. Klient może je pobierać i wypełniać.

```typescript
// Szablon promptu
{
  name: "summarize_thread",
  arguments: [{ name: "threadId", description: "ID wątku do podsumowania" }],
}
// Klient wywołuje: prompts/get z { threadId: "abc123" }
// Serwer zwraca: gotowy prompt z wypełnionym kontekstem
```

---

## 1.4 Jak wygląda komunikacja — transport

MCP definiuje protokół, ale nie transport. Ten szablon implementuje **Streamable HTTP**:

```
Klient                        Serwer
  │                              │
  │── POST /mcp (initialize) ──→ │  Ustanowienie sesji
  │←── 200 + Mcp-Session-Id ────│
  │                              │
  │── POST /mcp (tools/list) ──→│  Wywołanie synchroniczne
  │←── 200 + tools ─────────── │
  │                              │
  │── POST /mcp (tools/call) ──→│  Wywołanie z opcjonalnym streamingiem
  │←── 200 (lub SSE stream) ───│
  │                              │
  │── DELETE /mcp ─────────────→│  Zamknięcie sesji
  │←── 200 ─────────────────── │
```

**Co to jest SSE?** Server-Sent Events — jednokierunkowy strumień z serwera do klienta przez HTTP. Serwer może wysyłać wiele wiadomości na jedno żądanie. Używane do długich operacji z progress notifications i do bidirectional requests (sampling, elicitation).

**Dlaczego Workers nie obsługuje SSE?** Cloudflare Workers to funkcje wykonujące się w odpowiedzi na request. Nie mogą "żyć" między requestami i utrzymywać otwartego połączenia. Dlatego Workers obsługuje tylko request→response, bez streamingu.

---

## 1.5 Cykl życia sesji

Sesja MCP to nie to samo co sesja HTTP (cookie-based). To izolowany kontekst dla konkretnego klienta:

```
1. initialize (bez Mcp-Session-Id)
   → serwer tworzy sesję, zwraca ID w nagłówku

2. initialized (z Mcp-Session-Id)
   → serwer aktywuje sesję

3. tools/list, tools/call, resources/read...
   → każde żądanie waliduje ID sesji (404 jeśli nie istnieje)

4. DELETE /mcp LUB upłynięcie TTL (domyślnie 24h)
   → sesja usuwana
```

**Dlaczego sesje?** Multi-tenancy — jeden proces serwera obsługuje wielu użytkowników jednocześnie. Każdy ma swój izolowany kontekst (tokeny auth, ograniczenia zasobów).

---

## 1.6 Co znajdziesz w tym szablonie

Ten szablon to gotowa infrastruktura MCP. Implementuje:

- Cały cykl życia sesji zgodnie ze specyfikacją
- Pięć strategii autoryzacji
- Pełny OAuth 2.1 z PKCE
- Szyfrowane przechowywanie tokenów
- Dwa środowiska uruchomieniowe z jednej bazy kodu

Twoja praca to: **zdefiniować narzędzia** i **podłączyć klienta API** dostawcy, z którym się integrujesz.

---

## Podsumowanie

| Pojęcie | Znaczenie |
|---------|-----------|
| MCP | Protokół komunikacji model AI ↔ serwer zewnętrznych narzędzi |
| JSON-RPC 2.0 | Format wiadomości: `{ method, params, id }` |
| Tool | Funkcja wywoływana przez model AI |
| Resource | Dane tylko do odczytu identyfikowane URI |
| Prompt | Szablon wiadomości z parametrami |
| Sesja | Izolowany kontekst jednego klienta na serwerze |
| N×M problem | Bez wspólnego protokołu każda para (model × API) wymaga osobnej integracji |

**Dalej:** [02 — Architektura dualna: Node.js i Cloudflare Workers](./02-architektura-dualna.md)
