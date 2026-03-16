# 07 — Pomysły na serwery MCP: katalog integracji

> **Cel tego dokumentu:** Pokazać, jakie serwery MCP można zbudować na tym szablonie, czemu każdy z nich jest wartościowy i jakie funkcje szablonu są przy tym używane.
>
> **Format:** Każda integracja zawiera: opis problemu, propozycje narzędzi (tools), wymaganą strategię auth i wskazówki implementacyjne.

---

## Jak czytać ten katalog

Każdy wpis to gotowy pomysł na **oddzielny serwer MCP** — oddzielne repozytorium, oddzielny deployment. Szablon rozwiązuje powtarzalne problemy (auth, sesje, storage, dual-runtime), więc możesz skupić się wyłącznie na logice danego API.

Użyj `manual.md` (9 faz) jako checklisty dla każdej integracji.

---

## Kategoria 1: Produktywność i zarządzanie zadaniami

### 1.1 Linear MCP

**Problem:** Zarządzanie zadaniami w Linear wymaga klikania przez UI. Agent mógłby sam tworzyć zadania, przypisywać je i zmieniać statusy na podstawie rozmowy.

**Narzędzia do zaimplementowania:**

| Narzędzie | Opis | Hint |
|-----------|------|------|
| `list_issues` | Lista zadań według projektu/statusu/przypisanego | `readOnlyHint: true` |
| `create_issue` | Utwórz nowe zadanie z tytułem, opisem, priorytetem | `destructiveHint: false` |
| `update_issue` | Zmień status, przypisanego, priorytet | `idempotentHint: true` |
| `search_issues` | Wyszukaj zadania po słowie kluczowym lub filtrach | `readOnlyHint: true` |
| `get_issue` | Pobierz szczegóły zadania z komentarzami | `readOnlyHint: true` |
| `add_comment` | Dodaj komentarz do zadania | `destructiveHint: false` |
| `list_projects` | Lista projektów w workspace | `readOnlyHint: true` |
| `list_teams` | Lista zespołów i ich workflow | `readOnlyHint: true` |

**Auth:** `oauth` — Linear używa OAuth 2.0. RS token → Linear access token.

**SDK:** `@linear/sdk` — gotowy klient TypeScript, nie trzeba pisać fetch ręcznie.

**Funkcje szablonu:**
- `KvTokenStore` / `FileTokenStore` do mapowania RS → Linear token
- `defineTool()` + Zod do walidacji parametrów zadania
- Proaktywny refresh tokenu (`src/shared/oauth/refresh.ts`) — Linear tokeny wygasają

**Wskazówka:** Linear SDK zwraca obiekty z `Promise<T>` — używaj `async/await`. Paginacja działa przez `after` (cursor), co dobrze pasuje do mechanizmu `cursor` z szablonu.

---

### 1.2 Notion MCP

**Problem:** Bazy danych Notion to potężne narzędzie, ale odpytywanie ich i zapisywanie rekordów przez API wymaga znajomości struktury bloków.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `query_database` | Odpytaj bazę danych z filtrami i sortowaniem |
| `create_page` | Utwórz stronę w bazie danych |
| `update_page` | Zaktualizuj właściwości strony |
| `get_page` | Pobierz treść strony jako Markdown |
| `search_pages` | Szukaj stron po tytule lub treści |
| `append_blocks` | Dodaj bloki tekstu do strony |

**Auth:** `oauth` — Notion wymaga OAuth z zakresem `databases:read`, `pages:write`.

**Funkcje szablonu:**
- `MemorySessionStore` wystarczy dla Notion (sesje krótkie)
- Notion API zwraca bloki — warto napisać formatter do Markdown w `src/shared/utils/`
- `openWorldHint: true` dla narzędzi wyszukiwania (wyniki zmienne)

**Wskazówka:** Notion API ma limit 100 bloków na żądanie. Użyj mechanizmu paginacji szablonu (cursor → `start_cursor` w Notion API).

---

### 1.3 Jira MCP

**Problem:** Jira jest wszechobecna w korporacjach, ale API bywa nieczytelne. MCP może wystawić uproszczony interfejs.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_issues` | Lista zadań według JQL (Jira Query Language) |
| `create_issue` | Utwórz zadanie w wybranym projekcie |
| `transition_issue` | Zmień status zadania (np. "In Progress" → "Done") |
| `add_comment` | Dodaj komentarz |
| `get_sprint` | Informacje o aktywnym sprincie |
| `list_projects` | Lista projektów dostępnych dla użytkownika |

**Auth:** `oauth` — Jira Cloud wspiera OAuth 2.0 (3LO). Jira Server wymaga Basic Auth → użyj strategii `custom`.

**Funkcje szablonu:** Strategia `custom` (dla Jira Server z Basic Auth) pokazuje elastyczność `src/shared/auth/strategy.ts` — można przekazać dowolne nagłówki przez `CUSTOM_HEADERS`.

---

## Kategoria 2: Komunikacja i e-mail

### 2.1 Gmail MCP

**Problem:** Przetwarzanie e-maili, szukanie wiadomości i zarządzanie wątkami przez UI jest wolne. Agent może to zautomatyzować.

**Narzędzia:**

| Narzędzie | Opis | Hint |
|-----------|------|------|
| `list_threads` | Lista wątków z inbox/folder | `readOnlyHint: true` |
| `get_thread` | Treść wątku z wszystkimi wiadomościami | `readOnlyHint: true` |
| `search_threads` | Szukaj wątków (składnia Gmail: `from:`, `is:unread`) | `readOnlyHint: true` |
| `send_email` | Wyślij nową wiadomość | `destructiveHint: true` |
| `reply_to_thread` | Odpowiedz w wątku | `destructiveHint: true` |
| `create_draft` | Utwórz szkic bez wysyłania | `destructiveHint: false` |
| `label_thread` | Dodaj/usuń etykiety | `idempotentHint: true` |
| `archive_thread` | Archiwizuj wątek | `destructiveHint: false` |
| `get_labels` | Lista dostępnych etykiet | `readOnlyHint: true` |

**Auth:** `oauth` — Google wymaga `OAUTH_EXTRA_AUTH_PARAMS=access_type=offline&prompt=consent` żeby dostać refresh token.

**Zakresy:** `https://www.googleapis.com/auth/gmail.readonly` + `gmail.send` + `gmail.modify`

**Funkcje szablonu:**
- `FileTokenStore` z szyfrowaniem AES-256-GCM — tokeny Google są wrażliwe
- Elicitation (tylko Node.js): przed wysłaniem e-maila można zapytać użytkownika o potwierdzenie przez `src/shared/utils/elicitation.ts`
- Progress notifications dla długich operacji (`list_threads` z dużym limitem)

**Ważne:** `send_email` to `destructiveHint: true` — nieodwracalna operacja. Warto zaimplementować `create_draft` jako bezpieczniejszy krok pośredni i używać elicitation do potwierdzenia.

---

### 2.2 Slack MCP

**Problem:** Monitorowanie kanałów, wysyłanie powiadomień i czytanie historii przez agenta.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_channels` | Lista kanałów (publiczne + prywatne do których user należy) |
| `get_channel_history` | Historia wiadomości z kanału |
| `send_message` | Wyślij wiadomość na kanał lub do użytkownika |
| `reply_to_thread` | Odpowiedz w wątku Slack |
| `search_messages` | Szukaj wiadomości po treści |
| `list_users` | Lista użytkowników workspace |
| `get_user_profile` | Profil użytkownika (status, timezone) |
| `upload_file` | Wyślij plik na kanał |

**Auth:** `oauth` — Slack Bot Token przez OAuth 2.0. Alternatywnie `bearer` z Bot Token z env.

**Uwaga architektoniczna:** Slack oferuje też WebSocket Events API (real-time). To wykracza poza request/response MCP — ale narzędzia do odpytywania historii działają świetnie w modelu pull.

---

### 2.3 Microsoft Outlook MCP (Graph API)

**Problem:** Outlook/Exchange przez Microsoft Graph API to potężne narzędzie, szczególnie w środowiskach korporacyjnych.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_messages` | Lista wiadomości z folderu |
| `get_message` | Treść wiadomości |
| `send_message` | Wyślij e-mail |
| `list_calendar_events` | Nadchodzące spotkania |
| `create_calendar_event` | Utwórz spotkanie z uczestnikami |
| `list_contacts` | Lista kontaktów |

**Auth:** `oauth` — Azure AD OAuth 2.0 z PKCE. `PROVIDER_ACCOUNTS_URL=https://login.microsoftonline.com/{tenant}`.

**Funkcje szablonu:** PKCE jest wbudowane w szablon (`src/shared/oauth/flow.ts`) — Azure AD wymaga S256, co jest domyślne.

---

## Kategoria 3: Repozytoria i kod

### 3.1 GitHub MCP

**Problem:** Zarządzanie repozytoriami, issues i pull requestami bez opuszczania czatu.

**Narzędzia:**

| Narzędzie | Opis | Hint |
|-----------|------|------|
| `list_repos` | Lista repozytoriów użytkownika/organizacji | `readOnlyHint: true` |
| `get_repo` | Informacje o repozytorium | `readOnlyHint: true` |
| `list_issues` | Issues z filtrami (state, labels, assignee) | `readOnlyHint: true` |
| `create_issue` | Utwórz nowy issue | `destructiveHint: false` |
| `list_pull_requests` | Lista PR z filtrami | `readOnlyHint: true` |
| `get_pull_request` | Szczegóły PR z diff | `readOnlyHint: true` |
| `create_pull_request` | Utwórz PR | `destructiveHint: false` |
| `list_commits` | Historia commitów | `readOnlyHint: true` |
| `get_file_content` | Treść pliku z repozytorium | `readOnlyHint: true` |
| `search_code` | Szukaj kodu w GitHub | `readOnlyHint: true` |
| `list_workflow_runs` | Status GitHub Actions | `readOnlyHint: true` |
| `add_issue_comment` | Skomentuj issue | `destructiveHint: false` |

**Auth:** `oauth` (dla użytkownika) lub `bearer` (Personal Access Token z env) — dwa scenariusze, dwie strategie.

**Wskazówka:** Dla `bearer` z PAT: `AUTH_STRATEGY=bearer`, `BEARER_TOKEN=ghp_...`. Dla OAuth: pełny flow. Szablon obsługuje oba bez zmian w kodzie narzędzi — `context.providerToken` zawsze zwraca właściwy token.

---

### 3.2 GitLab MCP

Analogiczny do GitHub, ale z GitLab API. Różnica w auth: GitLab wspiera OAuth 2.0, ale też Personal Access Tokens. Zakresy: `api`, `read_user`, `read_repository`.

---

## Kategoria 4: Bazy danych i dane

### 4.1 Airtable MCP

**Problem:** Airtable to popularny "low-code database". MCP może wystawić jego tabele jako narzędzia.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_bases` | Lista baz danych Airtable |
| `list_tables` | Tabele w bazie |
| `list_records` | Rekordy z tabeli (z filtrami i sortowaniem) |
| `get_record` | Jeden rekord po ID |
| `create_record` | Utwórz nowy rekord |
| `update_record` | Zaktualizuj pola rekordu |
| `delete_record` | Usuń rekord |
| `search_records` | Szukaj przez formula filter |

**Auth:** `api_key` — Airtable używa Personal Access Token. `AUTH_STRATEGY=api_key`, `API_KEY_HEADER=Authorization` (format: `Bearer pat_...`).

**Funkcje szablonu:** Strategia `api_key` to najprostsza ścieżka — zero OAuth, zero sesji tokenowych. Idealny przykład do nauki zanim wejdziesz w OAuth.

---

### 4.2 Supabase MCP

**Problem:** Supabase jako backend-as-a-service ma REST API (PostgREST) do tabel i RPC do funkcji.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `query_table` | SELECT z filtrami (PostgREST syntax) |
| `insert_rows` | INSERT do tabeli |
| `update_rows` | UPDATE z filtrem |
| `delete_rows` | DELETE z filtrem |
| `call_function` | Wywołaj funkcję PostgreSQL przez RPC |
| `get_schema` | Schemat tabel (kolumny, typy) |

**Auth:** `bearer` lub `api_key` — Service Role Key jako statyczny token.

**Uwaga:** Daj agentowi tylko klucz `anon` (ograniczone uprawnienia) lub użyj Row Level Security. Service Role Key to dostęp admin — nie dawaj go agentowi do czytania ogólnych zapytań.

---

### 4.3 Google Sheets MCP

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_spreadsheets` | Lista arkuszy na Google Drive |
| `get_sheet_data` | Dane z zakresu (np. `Sheet1!A1:Z100`) |
| `update_cells` | Zapisz dane do zakresu |
| `append_rows` | Dodaj wiersze na końcu arkusza |
| `create_spreadsheet` | Nowy arkusz |
| `get_sheet_metadata` | Nazwy arkuszy, liczba wierszy |

**Auth:** `oauth` — Google OAuth z zakresem `spreadsheets` lub `spreadsheets.readonly`.

---

## Kategoria 5: Finanse i płatności

### 5.1 Stripe MCP

**Problem:** Monitorowanie płatności, klientów i subskrypcji bez logowania do dashboardu.

**Narzędzia:**

| Narzędzie | Opis | Hint |
|-----------|------|------|
| `list_customers` | Lista klientów z filtrami | `readOnlyHint: true` |
| `get_customer` | Szczegóły klienta | `readOnlyHint: true` |
| `list_payments` | Historia płatności | `readOnlyHint: true` |
| `get_payment` | Szczegóły transakcji | `readOnlyHint: true` |
| `list_subscriptions` | Aktywne subskrypcje | `readOnlyHint: true` |
| `list_invoices` | Faktury klienta | `readOnlyHint: true` |
| `create_refund` | Zwrot płatności | `destructiveHint: true` |

**Auth:** `bearer` lub `api_key` — Stripe Secret Key. **Uwaga bezpieczeństwa:** Użyj klucza z ograniczonym dostępem (Restricted Key) zamiast Secret Key — Stripe Dashboard pozwala tworzyć klucze tylko do odczytu.

**Wskazówka:** `create_refund` to operacja nieodwracalna (`destructiveHint: true`). Dobrym wzorcem jest elicitation (Node.js): zapytaj użytkownika o potwierdzenie przed wykonaniem zwrotu.

---

### 5.2 Xero MCP (księgowość)

**Problem:** Dostęp do faktur, kontaktów i raportów finansowych z poziomu czatu.

**Auth:** `oauth` — Xero używa OAuth 2.0 z PKCE. Tokeny wygasają po 30 minutach — proaktywny refresh szablonu jest tu kluczowy.

**Narzędzia:** `list_invoices`, `get_invoice`, `list_contacts`, `create_invoice`, `get_profit_loss_report`, `list_accounts`.

---

## Kategoria 6: Infrastruktura i DevOps

### 6.1 Cloudflare MCP

**Problem:** Zarządzanie strefami DNS, Workers i KV przez API zamiast dashboardu.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_zones` | Lista domen (stref DNS) |
| `list_dns_records` | Rekordy DNS dla strefy |
| `create_dns_record` | Dodaj rekord DNS |
| `update_dns_record` | Zaktualizuj rekord |
| `list_workers` | Lista Workers w koncie |
| `get_worker_script` | Kod Worker |
| `list_kv_namespaces` | Przestrzenie nazw KV |
| `list_kv_keys` | Klucze w namespace |
| `get_kv_value` | Wartość klucza KV |

**Auth:** `api_key` — Cloudflare API Token (`CF-Authorization` lub `Authorization: Bearer`).

**Wskazówka:** Ten serwer możesz wdrożyć jako Cloudflare Worker (korzystając z szablonu!) i zarządzać innymi Workerami przez MCP. Inception level: MCP Worker zarządzający Workerami.

---

### 6.2 Vercel MCP

**Narzędzia:** `list_projects`, `list_deployments`, `get_deployment_logs`, `trigger_deployment`, `list_env_vars`, `add_env_var`.

**Auth:** `bearer` — Vercel Access Token.

---

### 6.3 PagerDuty MCP

**Problem:** Sprawdzanie alertów, incydentów i dyżurów przez czat zamiast aplikacji mobilnej.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_incidents` | Aktywne incydenty |
| `get_incident` | Szczegóły incydentu z timeline |
| `acknowledge_incident` | Potwierdź incydent |
| `resolve_incident` | Oznacz jako rozwiązany |
| `list_on_call` | Kto jest na dyżurze |
| `list_services` | Lista serwisów |

**Auth:** `api_key` — PagerDuty REST API key.

---

## Kategoria 7: AI i dane

### 7.1 Pinecone MCP (vector database)

**Problem:** Przeszukiwanie baz wektorowych, zarządzanie indeksami i wektorami.

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `list_indexes` | Lista indeksów Pinecone |
| `query_index` | Szukaj przez wektor podobieństwa |
| `upsert_vectors` | Dodaj/zaktualizuj wektory |
| `delete_vectors` | Usuń wektory po ID |
| `get_index_stats` | Statystyki indeksu |

**Auth:** `api_key` — Pinecone API Key.

**Ciekawy case:** Możesz połączyć sampling (Node.js) z Pinecone — agent pobiera tekst, wysyła do LLM przez sampling żeby uzyskać embedding, potem odpytuje Pinecone. To przykład server→client→LLM flow.

---

### 7.2 Perplexity / Brave Search MCP

**Problem:** Dawanie agentowi możliwości przeszukiwania internetu.

**Narzędzia:** `search`, `get_page_content`.

**Auth:** `api_key`.

**Funkcje szablonu:** Najprostszy możliwy serwer MCP — dwa narzędzia, `api_key` auth, zero OAuth. Idealny jako pierwsze ćwiczenie.

---

## Kategoria 8: CRM i marketing

### 8.1 HubSpot MCP

**Narzędzia:**

| Narzędzie | Opis |
|-----------|------|
| `search_contacts` | Szukaj kontaktów |
| `get_contact` | Szczegóły kontaktu z historią |
| `create_contact` | Nowy kontakt |
| `update_contact` | Zaktualizuj właściwości |
| `list_deals` | Lista transakcji (deals) |
| `create_deal` | Nowa transakcja |
| `get_company` | Szczegóły firmy |
| `list_tickets` | Zgłoszenia helpdesk |

**Auth:** `oauth` — HubSpot OAuth 2.0 z zakresami per obiekt (contacts, crm.objects.deals.read, etc.).

---

### 8.2 Mailchimp MCP

**Narzędzia:** `list_campaigns`, `get_campaign_stats`, `list_audiences`, `get_audience_members`, `add_member`, `send_campaign`.

**Auth:** `oauth` — Mailchimp OAuth 2.0. Uwaga: endpoint API zależy od `data_center` z profilu użytkownika (np. `us1.api.mailchimp.com`) — trzeba go pobrać po auth.

---

## Jak wybrać co zbudować

### Macierz trudności vs wartości

```
Trudność ↑
         │
  wysoka │  Outlook (Azure AD)    Gmail (tokeny Google)
         │  Xero (krótkie tokeny) Jira Server (Basic Auth)
         │
 średnia │  GitHub (oauth/bearer) Linear (SDK)
         │  HubSpot (zakresy)     Notion (bloki)
         │
   niska │  Airtable (api_key)    Stripe (bearer)
         │  Perplexity (api_key)  Cloudflare (api_key)
         └────────────────────────────────────────────→ Wartość
              niska          średnia          wysoka
```

### Rekomendowana ścieżka nauki

**Krok 1 — Zacznij od `api_key`:**
Airtable lub Perplexity. Zero OAuth, jeden token, dwa narzędzia. Poznasz `defineTool()`, `asRegisteredTool()`, registry.

**Krok 2 — Dodaj OAuth:**
GitHub lub Linear. Poznasz pełny flow OAuth, mapowanie RS token → provider token, refresh.

**Krok 3 — Złożona integracja:**
Gmail lub HubSpot. Wiele narzędzi, wiele zakresów OAuth, elicitation przy destrukcyjnych operacjach.

---

## Wzorce wspólne dla wszystkich integracji

### Wzorzec: token z kontekstu

Wszystkie narzędzia dostają token identycznie, niezależnie od strategii auth:

```typescript
// Zawsze tak samo — tools nie wiedzą skąd pochodzi token
handler: async (args, context) => {
  const token = context.providerToken;  // lub context.provider?.accessToken
  if (!token) {
    return { isError: true, content: [{ type: 'text', text: 'Wymagane logowanie.' }] };
  }
  // ...
}
```

### Wzorzec: LLM-friendly errors

Nie rzucaj technicznych błędów do LLM. Tłumacz na ludzki język:

```typescript
} catch (error) {
  const msg = error instanceof Error ? error.message : String(error);
  // Zamiast: "TypeError: Cannot read properties of undefined"
  // Daj: "Nie udało się pobrać zadań. Sprawdź czy masz dostęp do projektu."
  return { isError: true, content: [{ type: 'text', text: `Błąd: ${msg}` }] };
}
```

### Wzorzec: Paginacja przez cursor

Wszystkie listy powinny obsługiwać paginację. Szablon dostarcza mechanizm, musisz go zmapować na API:

```typescript
// args.cursor to twój cursor z poprzedniej odpowiedzi
const apiParams = {
  limit: args.limit ?? 25,
  // GitHub: page_token, Linear: after, Notion: start_cursor
  after: args.cursor,
};

// W odpowiedzi:
return {
  content: [{ type: 'text', text: formatList(items) }],
  structuredContent: {
    items,
    pagination: {
      hasMore: !!nextCursor,
      nextCursor,  // klient użyje tego jako args.cursor w następnym wywołaniu
    },
  },
};
```

### Wzorzec: Annotations jako kontrakt

Annotations to informacja dla klienta MCP jak zachować się względem narzędzia:

```typescript
annotations: {
  readOnlyHint: true,      // to narzędzie nic nie modyfikuje
  destructiveHint: false,  // operacja odwracalna
  idempotentHint: true,    // wielokrotne wywołanie = ten sam efekt
  openWorldHint: false,    // wyniki deterministyczne (nie external web)
}
```

Klient (np. Claude Desktop) może użyć tych wskazówek żeby np. zawsze pytać o zgodę przed `destructiveHint: true`.

---

## Szybkie porównanie strategii auth

| Strategia | Kiedy używać | Przykładowe API | Konfiguracja |
|-----------|-------------|-----------------|--------------|
| `none` | Wewnętrzne narzędzia bez auth | Lokalny skrypt, dev | `AUTH_STRATEGY=none` |
| `api_key` | API z jednym kluczem | Airtable, Stripe, Pinecone | `AUTH_STRATEGY=api_key`, `API_KEY=...` |
| `bearer` | Statyczny token dostępu | GitHub PAT, Vercel, Supabase | `AUTH_STRATEGY=bearer`, `BEARER_TOKEN=...` |
| `custom` | Wiele nagłówków, niestandardowe | Jira Server, stare API | `AUTH_STRATEGY=custom`, `CUSTOM_HEADERS=...` |
| `oauth` | Użytkownik loguje się sam | Linear, Gmail, GitHub OAuth | Pełna konfiguracja OAuth |

---

## Podsumowanie

Ten szablon rozwiązuje najtrudniejsze problemy raz — OAuth 2.1, PKCE, podwójny token, szyfrowanie, sesje, dual-runtime. Twoja praca przy każdej nowej integracji to:

1. **Wybrać strategię auth** (5 minut decyzji)
2. **Zdefiniować narzędzia** (schemas + opisy)
3. **Napisać klienta API** (fetch lub SDK)
4. **Zarejestrować w registry** (jedna linijka)

Reszta działa automatycznie.

---

*Poprzedni dokument: [06-jak-dodac-tool.md](./06-jak-dodac-tool.md)*
