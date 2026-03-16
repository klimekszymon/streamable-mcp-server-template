# 08 — Narzędzia edukacyjne (mocki): testowanie możliwości i ograniczeń MCP

> **Cel tego dokumentu:** Pokazać zestaw prostych narzędzi-mocków, które nie wymagają żadnych zewnętrznych usług ani kluczy API. Każde z nich ilustruje inny wzorzec projektowy lub ograniczenie protokołu MCP, zmuszając agenta do radzenia sobie z konkretnymi sytuacjami.
>
> **Format:** Każde narzędzie zawiera: nazwę, logikę mocka (co zwraca), czego uczy oraz propozycję schematu Zod.
>
> **Jak uruchomić:** Zaimplementuj każde narzędzie w `src/shared/tools/` i zarejestruj w `src/shared/tools/registry.ts`. Następnie użyj `bun run mcp:inspector` lub podłącz klienta MCP, by obserwować zachowanie agenta.

---

## Dlaczego mocki, a nie prawdziwe API?

Prawdziwe integracje wymagają kont, kluczy i konfiguracji. Mocki pozwalają skupić się wyłącznie na **zachowaniu agenta** — jak reaguje na paginację, błędy, wskazówki, ograniczenia i asynchroniczność. Możesz je wdrożyć w kilka minut i od razu testować hipotezy dotyczące projektowania interfejsów narzędzi.

Zasada: **każde narzędzie uczy jednej rzeczy**.

---

## 1. Zarządzanie wielkością kontekstu (Paginacja)

**Nazwa:** `api__get_records_mock`

**Czego uczy:** Paginacja i nawigacja po wynikach są kluczowe — agenci muszą unikać pobierania zbyt dużej ilości danych naraz, bo zatyka to okno kontekstu. Testujesz tu, czy agent po przeczytaniu hintu wywoła narzędzie ponownie z odpowiednim parametrem `page`.

**Logika mocka:**

Przyjmuje opcjonalne parametry `limit` i `page`. Jeśli agent nie poda `limit` lub poda wartość większą niż 5, narzędzie zwraca tylko 5 pustych obiektów i dokleja dynamiczny hint. Jeśli agent poprawnie poda `limit <= 5`, zwraca żądaną stronę.

```typescript
export const apiGetRecordsMock = defineTool({
  name: 'api__get_records_mock',
  description: 'Pobiera rekordy z fikcyjnej bazy danych. Obsługuje paginację.',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    limit: z.number().optional().describe('Maksymalna liczba rekordów (max 5)'),
    page:  z.number().optional().describe('Numer strony (domyślnie 1)'),
  }),
  handler: async ({ limit = 50, page = 1 }) => {
    const PAGE_SIZE = 5;
    const TOTAL    = 50;
    const effective = Math.min(limit, PAGE_SIZE);
    const records   = Array.from({ length: effective }, (_, i) => ({
      id: `rec_${(page - 1) * PAGE_SIZE + i + 1}`,
      data: {},
    }));

    const hint =
      limit > PAGE_SIZE
        ? `Zwrócono tylko ${PAGE_SIZE} wyników z ${TOTAL}. Użyj parametru page=${page + 1}, aby pobrać kolejne.`
        : page * PAGE_SIZE < TOTAL
        ? `Strona ${page} z ${Math.ceil(TOTAL / PAGE_SIZE)}. Użyj page=${page + 1} po więcej.`
        : 'To jest ostatnia strona wyników.';

    return {
      content: [{
        type: 'text' as const,
        text: JSON.stringify({ records, hints: hint }, null, 2),
      }],
    };
  },
});
```

**Obserwuj:** Czy agent wywołuje narzędzie ponownie z `page=2`? Czy rozumie, że `limit` powyżej 5 jest ignorowany?

---

## 2. Bezpieczne modyfikacje i testowanie na sucho (Dry Run)

**Nazwa:** `fs__write_safe_mock`

**Czego uczy:** Modyfikowanie plików stwarza przestrzeń do błędów. Flaga `dryRun` pozwala sprawdzić efekt przed faktycznym zapisem. Dodatkowa weryfikacja `checksum` chroni przed nadpisaniem pliku zmienionego w międzyczasie — testujesz, czy agent samodzielnie wykonuje dwuetapowy flow: najpierw dry run, potem zapis z checksumą.

**Logika mocka:**

Jeśli `dryRun === true` — nie "zapisuje", zwraca podgląd. Jeśli `dryRun` nie jest podane, a brak `checksum` — rzuca błąd z instrukcją. Jeśli checksum nie pasuje do oczekiwanej — blokuje zapis (symulacja konfliktu).

```typescript
const MOCK_CHECKSUM = 'abc123';

export const fsWriteSafeMock = defineTool({
  name: 'fs__write_safe_mock',
  description: 'Zapisuje treść do fikcyjnego pliku. Wymaga dry run przed faktycznym zapisem.',
  annotations: { destructiveHint: false, idempotentHint: false },
  inputSchema: z.object({
    content:  z.string().describe('Treść do zapisania'),
    dryRun:   z.boolean().optional().describe('Jeśli true, tylko symuluje zapis'),
    checksum: z.string().optional().describe('Checksum aktualnej wersji pliku (wymagany przy zapisie)'),
  }),
  handler: async ({ content, dryRun, checksum }) => {
    if (dryRun) {
      return {
        content: [{
          type: 'text' as const,
          text: `Sukces (Dry Run). Dokument po zmianach miałby ${content.length} znaków. Aktualna checksum pliku: "${MOCK_CHECKSUM}". Jeśli wszystko się zgadza, uruchom ponownie bez flagi dryRun, podając checksum="${MOCK_CHECKSUM}".`,
        }],
      };
    }
    if (!checksum) {
      return {
        isError: true,
        content: [{
          type: 'text' as const,
          text: 'Błąd: brak parametru checksum. Najpierw wykonaj wywołanie z dryRun=true, aby uzyskać checksumę aktualnej wersji pliku.',
        }],
      };
    }
    if (checksum !== MOCK_CHECKSUM) {
      return {
        isError: true,
        content: [{
          type: 'text' as const,
          text: `Błąd konfliktu: plik został zmieniony od ostatniego odczytu (oczekiwano checksum="${MOCK_CHECKSUM}", otrzymano "${checksum}"). Odczytaj plik ponownie przed zapisem.`,
        }],
      };
    }
    return {
      content: [{
        type: 'text' as const,
        text: `Sukces. Zapisano ${content.length} znaków. Nowa checksum: "def456".`,
      }],
    };
  },
});
```

**Obserwuj:** Czy agent wykona dry run zanim spróbuje zapisać? Czy przekaże checksum? Czy zrozumie komunikat o konflikcie?

---

## 3. Korekty wprowadzane przez narzędzia (Inteligentne błędy)

**Nazwa:** `fs__read_lines_mock`

**Czego uczy:** Informowania agenta o korektach wprowadzonych przez narzędzie i radzenia sobie z nieścisłościami wynikającymi z halucynacji modelu (np. błędna długość dokumentu). Obserwujesz, czy model rozumie, że dotarł do końca pliku i nie próbuje czytać dalej.

**Logika mocka:**

Fikcyjny dokument ma zawsze 59 linii. Jeśli agent poprosi o zakres wykraczający poza 59, narzędzie zwraca maksymalny dostępny zakres z informacją o korekcie.

```typescript
const MOCK_TOTAL_LINES = 59;
const MOCK_LINES = Array.from(
  { length: MOCK_TOTAL_LINES },
  (_, i) => `Linia ${i + 1}: Lorem ipsum dolor sit amet.`
);

export const fsReadLinesMock = defineTool({
  name: 'fs__read_lines_mock',
  description: 'Czyta zakres linii z fikcyjnego dokumentu (dokument ma 59 linii).',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    startLine: z.number().int().min(1).describe('Pierwsza linia do wczytania'),
    endLine:   z.number().int().describe('Ostatnia linia do wczytania'),
  }),
  handler: async ({ startLine, endLine }) => {
    const clampedEnd  = Math.min(endLine, MOCK_TOTAL_LINES);
    const clampedStart = Math.max(startLine, 1);
    const lines = MOCK_LINES.slice(clampedStart - 1, clampedEnd);
    const correctionNote =
      endLine > MOCK_TOTAL_LINES
        ? `\n\n[Uwaga: Żądano zakresu ${startLine}–${endLine}, ale dokument ma tylko ${MOCK_TOTAL_LINES} linii. Wczytano dostępny zakres ${clampedStart}–${clampedEnd}.]`
        : '';
    return {
      content: [{
        type: 'text' as const,
        text: lines.join('\n') + correctionNote,
      }],
    };
  },
});
```

**Obserwuj:** Czy model po korekcie rozumie, że dotarł do końca i przestaje prosić o więcej? Czy halucynuje liczbę linii w kolejnych wywołaniach?

---

## 4. Optymalizacja interfejsu (Konsolidacja akcji)

**Nazwa:** `fs__manage_mock`

**Czego uczy:** Jedno narzędzie z parametrem `action` może zastąpić wiele osobnych endpointów, zmniejszając liczbę schematów do zapamiętania przez agenta. Podwójny podkreślnik (`fs__`) symuluje zapobieganie konfliktom nazw między serwerami MCP. Błąd z hintem przy niepustym katalogu uczy, że narzędzia mogą prowadzić agenta przez wieloetapowe operacje.

**Logika mocka:**

Jedno narzędzie obsługuje: `create_dir`, `delete_file`, `move_file`, `list_dir`. Przy próbie usunięcia niepustego katalogu (symulowanego przez konkretną nazwę) zwraca błąd z podpowiedzią.

```typescript
export const fsManageMock = defineTool({
  name: 'fs__manage_mock',
  description: 'Zarządza plikami i katalogami. Akcje: create_dir, delete_file, move_file, list_dir.',
  annotations: { destructiveHint: false },
  inputSchema: z.object({
    action: z.enum(['create_dir', 'delete_file', 'move_file', 'list_dir']),
    path:   z.string().describe('Ścieżka docelowa'),
    dest:   z.string().optional().describe('Ścieżka docelowa (tylko dla move_file)'),
  }),
  handler: async ({ action, path, dest }) => {
    if (action === 'delete_file' && path === '/mock/non-empty-dir') {
      return {
        isError: true,
        content: [{
          type: 'text' as const,
          text: `Błąd: Nie można usunąć "${path}" — katalog nie jest pusty. Najpierw usuń zawartość używając action="list_dir" aby zobaczyć pliki, a następnie usuń je jeden po drugim.`,
        }],
      };
    }
    const results: Record<string, string> = {
      create_dir:  `Utworzono katalog: ${path}`,
      delete_file: `Usunięto: ${path}`,
      move_file:   `Przeniesiono: ${path} → ${dest ?? '(brak dest)'}`,
      list_dir:    `Zawartość ${path}:\n- plik_a.txt\n- plik_b.txt\n- podkatalog/`,
    };
    return {
      content: [{ type: 'text' as const, text: results[action] }],
    };
  },
});
```

**Obserwuj:** Czy agent korzysta z jednego narzędzia dla różnych operacji? Czy rozumie komunikat o niepustym katalogu i wywołuje `list_dir` przed `delete_file`?

---

## 5. Symulacja asynchroniczności i Rate Limitów

**Nazwy:** `api__trigger_job_mock` + `api__check_job_mock`

**Czego uczy:** Mechanizmy pollingu i rate limitingu mogą być problematyczne dla agentów. Observujesz, czy agent: zatrzyma się i poczeka, powiadomi użytkownika o konieczności oczekiwania, czy wpadnie w pętlę ciągłych zapytań.

**Logika mocka:**

`api__trigger_job_mock` zawsze zwraca status `pending` z instrukcją. `api__check_job_mock` śledzi czas wywołania (przez prostą flagę w module) — jeśli wywołano za szybko, zwraca błąd rate limit; po "odczekaniu" (symulowanym przez flagę) zwraca `completed`.

```typescript
let jobStartedAt: number | null = null;

export const apiTriggerJobMock = defineTool({
  name: 'api__trigger_job_mock',
  description: 'Uruchamia długotrwały proces asynchroniczny.',
  annotations: { destructiveHint: false },
  inputSchema: z.object({
    jobType: z.string().describe('Typ zadania do uruchomienia'),
  }),
  handler: async ({ jobType }) => {
    jobStartedAt = Date.now();
    return {
      content: [{
        type: 'text' as const,
        text: `Proces "${jobType}" rozpoczęty. Status: pending. Sprawdź wynik za minimum 10 sekund używając narzędzia api__check_job_mock z jobId="mock-job-001".`,
      }],
    };
  },
});

export const apiCheckJobMock = defineTool({
  name: 'api__check_job_mock',
  description: 'Sprawdza status uruchomionego procesu asynchronicznego.',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    jobId: z.string().describe('ID zadania zwrócone przez api__trigger_job_mock'),
  }),
  handler: async ({ jobId }) => {
    if (!jobStartedAt) {
      return {
        isError: true,
        content: [{ type: 'text' as const, text: `Błąd: Nie znaleziono zadania "${jobId}". Najpierw uruchom api__trigger_job_mock.` }],
      };
    }
    const elapsed = Date.now() - jobStartedAt;
    if (elapsed < 10_000) {
      return {
        isError: true,
        content: [{
          type: 'text' as const,
          text: `Zbyt wcześnie. Rate limit przekroczony. Minęło tylko ${Math.round(elapsed / 1000)}s. Odczekaj dokładnie ${Math.round((10_000 - elapsed) / 1000)} sekund i spróbuj ponownie.`,
        }],
      };
    }
    return {
      content: [{
        type: 'text' as const,
        text: `Zadanie "${jobId}" zakończone sukcesem. Wynik: { "processedRows": 1337, "duration": "${Math.round(elapsed / 1000)}s" }`,
      }],
    };
  },
});
```

**Obserwuj:** Czy agent informuje użytkownika o konieczności czekania? Czy sam poczeka (timeout)? Czy zapętli się z polling co sekundę?

---

## 6. Ustrukturyzowane vs tekstowe odpowiedzi (Structured Content)

**Nazwa:** `data__get_report_mock`

**Czego uczy:** MCP pozwala zwracać zarówno `content` (czytelny dla człowieka tekst) jak i `structuredContent` (ustrukturyzowany JSON dla klienta). Testujesz, czy agent korzysta z danych strukturalnych do dalszego przetwarzania zamiast parsować tekst.

**Logika mocka:**

Zwraca fikcyjny raport sprzedażowy jednocześnie jako tekst (dla prezentacji) i jako obiekt JSON (dla dalszego przetwarzania). Tekst celowo pomija część pól obecnych w JSON, by sprawdzić, których danych agent używa.

```typescript
export const dataGetReportMock = defineTool({
  name: 'data__get_report_mock',
  description: 'Pobiera raport sprzedażowy za wybrany miesiąc.',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    month: z.string().describe('Miesiąc w formacie YYYY-MM'),
  }),
  handler: async ({ month }) => {
    const report = {
      month,
      totalRevenue: 48_250,
      currency: 'PLN',
      topProduct: { name: 'Widget Pro', units: 312, revenue: 18_720 },
      newCustomers: 47,
      churnRate: 0.023,
      rawData: Array.from({ length: 5 }, (_, i) => ({
        productId: `prod_${i + 1}`,
        units: Math.floor(Math.random() * 100) + 10,
      })),
    };

    return {
      content: [{
        type: 'text' as const,
        text: `Raport za ${month}: przychód ${report.totalRevenue} PLN, nowi klienci: ${report.newCustomers}, najlepszy produkt: ${report.topProduct.name}.`,
      }],
      // structuredContent dostępne w MCP 2025-03-26+
      // structuredContent: report,
    };
  },
});
```

**Obserwuj:** Czy agent pyta o dane niedostępne w tekście (np. `churnRate`, `rawData`)? Czy samodzielnie prosi o format JSON zamiast tekstu?

---

## 7. Walidacja i korekta parametrów wejściowych (Schema Enforcement)

**Nazwa:** `search__query_mock`

**Czego uczy:** Dobrze zaprojektowany schemat Zod z opisami zmusza agenta do podawania poprawnych parametrów i redukuje liczbę błędnych wywołań. Testujesz, jak agent radzi sobie z enumerowanymi wartościami i wzajemnie wykluczającymi się parametrami.

**Logika mocka:**

Przyjmuje `query`, opcjonalne `filters` (z enum na `status`) i `sortBy`. Jeśli agent poda nieznany `status`, Zod odrzuci wywołanie. Jeśli poda jednocześnie `fullText: true` i `exactMatch: true`, narzędzie zwraca błąd logiczny z wyjaśnieniem.

```typescript
export const searchQueryMock = defineTool({
  name: 'search__query_mock',
  description: 'Wyszukuje rekordy według zapytania i filtrów.',
  annotations: { readOnlyHint: true, openWorldHint: false },
  inputSchema: z.object({
    query:      z.string().min(1).describe('Tekst wyszukiwania'),
    filters: z.object({
      status:    z.enum(['active', 'archived', 'draft']).optional(),
      dateFrom:  z.string().optional().describe('ISO date, np. 2025-01-01'),
      dateTo:    z.string().optional().describe('ISO date, np. 2025-12-31'),
    }).optional(),
    sortBy:     z.enum(['relevance', 'date_asc', 'date_desc']).default('relevance'),
    fullText:   z.boolean().optional().describe('Szukaj w pełnej treści (wolniejsze)'),
    exactMatch: z.boolean().optional().describe('Wymagaj dokładnego dopasowania frazy'),
  }),
  handler: async ({ query, filters, sortBy, fullText, exactMatch }) => {
    if (fullText && exactMatch) {
      return {
        isError: true,
        content: [{
          type: 'text' as const,
          text: 'Błąd: parametry fullText i exactMatch wykluczają się wzajemnie. Użyj tylko jednego z nich.',
        }],
      };
    }
    const results = Array.from({ length: 3 }, (_, i) => ({
      id:      `result_${i + 1}`,
      title:   `Wynik ${i + 1} dla "${query}"`,
      status:  filters?.status ?? 'active',
      score:   (0.99 - i * 0.1).toFixed(2),
    }));
    return {
      content: [{
        type: 'text' as const,
        text: JSON.stringify({ results, meta: { query, sortBy, total: 3 } }, null, 2),
      }],
    };
  },
});
```

**Obserwuj:** Czy agent dostosowuje parametry po błędzie walidacji Zod? Czy rozumie komunikat o wzajemnym wykluczeniu?

---

## 8. Elicitation — pytanie użytkownika w trakcie wykonania (tylko Node.js)

**Nazwa:** `action__confirm_mock`

**Czego uczy:** Elicitation to mechanizm MCP pozwalający serwerowi zapytać użytkownika o dodatkowe dane lub potwierdzenie w trakcie wykonania narzędzia (bez przerywania flow przez agenta). Dostępne tylko na Node.js (wymaga SSE). Testujesz, jak agent obsługuje sytuację, gdy narzędzie samo prosi o dane zamiast polegać na parametrach.

**Logika mocka:**

Przed "wykonaniem" destrukcyjnej operacji narzędzie wywołuje `elicit()` z pytaniem o potwierdzenie. Jeśli użytkownik odmówi — operacja jest anulowana. Jeśli potwierdzi — "wykonuje się".

```typescript
import { elicit } from '../utils/elicitation.js';

export const actionConfirmMock = defineTool({
  name: 'action__confirm_mock',
  description: 'Wykonuje destrukcyjną operację po uzyskaniu potwierdzenia od użytkownika.',
  annotations: { destructiveHint: true },
  inputSchema: z.object({
    target:    z.string().describe('Obiekt do usunięcia'),
    requestId: z.string().optional(),
  }),
  handler: async ({ target }, context) => {
    // Elicitation — dostępne tylko na Node.js przez SSE
    try {
      const answer = await elicit(context.requestId ?? 'unknown', {
        message: `Czy na pewno chcesz usunąć "${target}"? Tej operacji nie można cofnąć.`,
        requestedSchema: {
          type: 'object',
          properties: {
            confirmed: { type: 'boolean', description: 'Wpisz true aby potwierdzić' },
          },
          required: ['confirmed'],
        },
      });
      if (!answer?.confirmed) {
        return {
          content: [{ type: 'text' as const, text: `Operacja anulowana przez użytkownika.` }],
        };
      }
    } catch {
      // Fallback jeśli elicitation niedostępne (Workers)
      return {
        isError: true,
        content: [{
          type: 'text' as const,
          text: 'Elicitation niedostępne w tym środowisku (wymaga Node.js + SSE). Wywołaj narzędzie z parametrem confirmed=true jeśli chcesz kontynuować.',
        }],
      };
    }
    return {
      content: [{ type: 'text' as const, text: `Usunięto "${target}". Operacja zakończona.` }],
    };
  },
});
```

**Obserwuj:** Czy agent pauzuje i czeka na odpowiedź użytkownika? Czy rozumie, że elicitation to nie błąd lecz żądanie danych?

---

## 9. Granularne komunikaty błędów (LLM-friendly Errors)

**Nazwa:** `debug__error_types_mock`

**Czego uczy:** Różne typy błędów powinny zwracać różne komunikaty — techniczny stack trace jest bezużyteczny dla agenta, ale precyzyjny opis błędu z sugestią naprawy pozwala na autokorektę. Testujesz, jak agent reaguje na różne kategorie błędów.

**Logika mocka:**

Na podstawie parametru `scenario` narzędzie symuluje różne kategorie błędów: `not_found`, `permission_denied`, `rate_limited`, `validation`, `server_error` — każdy z innym formatem komunikatu i inną sugestią działania.

```typescript
export const debugErrorTypesMock = defineTool({
  name: 'debug__error_types_mock',
  description: 'Symuluje różne typy błędów. Używane do testowania zachowania agenta.',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    scenario: z.enum([
      'not_found',
      'permission_denied',
      'rate_limited',
      'validation',
      'server_error',
      'success',
    ]).describe('Typ scenariusza do zasymulowania'),
    resourceId: z.string().optional(),
  }),
  handler: async ({ scenario, resourceId = 'res_42' }) => {
    const errors: Record<string, { isError: boolean; text: string }> = {
      not_found: {
        isError: true,
        text: `Zasób "${resourceId}" nie istnieje. Sprawdź ID zasobu lub użyj narzędzia search__query_mock, aby znaleźć właściwe ID.`,
      },
      permission_denied: {
        isError: true,
        text: `Brak uprawnień do zasobu "${resourceId}". Użytkownik nie ma roli "admin". Poproś administratora o nadanie dostępu lub użyj zasobu z własnego workspace.`,
      },
      rate_limited: {
        isError: true,
        text: `Rate limit przekroczony. Dozwolone: 10 req/min. Odczekaj 60 sekund przed następnym wywołaniem.`,
      },
      validation: {
        isError: true,
        text: `Błąd walidacji: pole "email" ma nieprawidłowy format. Oczekiwano formatu user@domain.com, otrzymano "not-an-email".`,
      },
      server_error: {
        isError: true,
        text: `Błąd serwera (500). Operacja nie powiodła się z powodu tymczasowej awarii. Spróbuj ponownie za 30 sekund. Jeśli problem persystuje, zgłoś błąd z ID: err_${Date.now()}.`,
      },
      success: {
        isError: false,
        text: `Zasób "${resourceId}" znaleziony. Status: aktywny, typ: document, rozmiar: 4.2KB.`,
      },
    };

    const result = errors[scenario];
    return {
      isError: result.isError,
      content: [{ type: 'text' as const, text: result.text }],
    };
  },
});
```

**Obserwuj:** Czy agent po `not_found` automatycznie próbuje wyszukać zasób? Czy po `rate_limited` czeka, czy od razu próbuje ponownie? Czy po `server_error` pyta użytkownika o instrukcje?

---

## 10. Progresywne ujawnianie informacji (Progressive Disclosure)

**Nazwa:** `docs__get_article_mock`

**Czego uczy:** Narzędzia mogą zwracać dane na różnych poziomach szczegółowości. Zamiast zawsze zwracać pełną treść (duże zużycie kontekstu), agent powinien najpierw poprosić o podsumowanie, a dopiero w razie potrzeby — o pełną wersję. Testujesz strategię "czytaj nagłówki przed treścią".

**Logika mocka:**

Parametr `detail` kontroluje granularność odpowiedzi: `summary` (kilka zdań + TOC), `section` (konkretna sekcja), `full` (cały dokument). Przy `full` bez podania `confirmedLargeResponse: true` zwraca ostrzeżenie.

```typescript
export const docsGetArticleMock = defineTool({
  name: 'docs__get_article_mock',
  description: 'Pobiera artykuł z dokumentacji na różnych poziomach szczegółowości.',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    articleId:             z.string(),
    detail:                z.enum(['summary', 'section', 'full']).default('summary'),
    section:               z.string().optional().describe('Nazwa sekcji (wymagane gdy detail=section)'),
    confirmedLargeResponse: z.boolean().optional().describe('Wymagane przy detail=full'),
  }),
  handler: async ({ articleId, detail, section, confirmedLargeResponse }) => {
    if (detail === 'full' && !confirmedLargeResponse) {
      return {
        content: [{
          type: 'text' as const,
          text: `Artykuł "${articleId}" ma 8500 znaków (ok. 2100 tokenów). Pobranie pełnej treści może zająć znaczną część kontekstu. Spis treści:\n1. Wprowadzenie\n2. Instalacja\n3. Konfiguracja\n4. Przykłady użycia\n5. FAQ\n\nAby pobrać pełny artykuł, wywołaj ponownie z confirmedLargeResponse=true. Aby pobrać wybraną sekcję, użyj detail="section" z parametrem section="Instalacja".`,
        }],
      };
    }
    if (detail === 'section') {
      if (!section) {
        return {
          isError: true,
          content: [{ type: 'text' as const, text: 'Parametr section jest wymagany gdy detail="section".' }],
        };
      }
      return {
        content: [{
          type: 'text' as const,
          text: `## ${section}\n\nTreść sekcji "${section}" artykułu "${articleId}".\n\nLorem ipsum dolor sit amet, consectetur adipiscing elit. Sekcja zawiera 3 podpunkty i 2 przykłady kodu.`,
        }],
      };
    }
    if (detail === 'summary') {
      return {
        content: [{
          type: 'text' as const,
          text: `Artykuł: "${articleId}"\nPodsumowanie: Dokument opisuje instalację i konfigurację narzędzia w środowisku produkcyjnym. Obejmuje 5 sekcji, 2 przykłady kodu i FAQ z 8 pytaniami.\nOstatnia aktualizacja: 2025-03-01`,
        }],
      };
    }
    // full + confirmed
    return {
      content: [{
        type: 'text' as const,
        text: `# ${articleId}\n\n${'Lorem ipsum dolor sit amet. '.repeat(200)}`,
      }],
    };
  },
});
```

**Obserwuj:** Czy agent zaczyna od `summary`? Czy przy `full` odczyta ostrzeżenie i użyje `section` zamiast pobierać całość? Czy samodzielnie nawiguje po TOC?

---

## 11. Annotations w praktyce (Tool Hints)

**Nazwa:** `ops__delete_resource_mock`

**Czego uczy:** Annotations (`destructiveHint`, `idempotentHint`) to sygnały dla klienta MCP o charakterze operacji. Klient (np. Claude Desktop) może automatycznie pytać o zgodę przed narzędziami z `destructiveHint: true`. Testujesz, czy klient MCP respektuje te wskazówki.

**Logika mocka:**

Dwa bliźniacze narzędzia wykonujące identyczną "operację", ale z różnymi annotations — jedno destrukcyjne (`destructiveHint: true`), drugie bezpieczne (`destructiveHint: false`, `idempotentHint: true`). Obserwuj różnicę w zachowaniu klienta.

```typescript
// Wersja destrukcyjna — klient POWINIEN pytać o zgodę
export const opsDeleteResourceMock = defineTool({
  name: 'ops__delete_resource_mock',
  description: 'Usuwa zasób trwale. Operacja nieodwracalna.',
  annotations: {
    destructiveHint: true,
    idempotentHint:  false,
    readOnlyHint:    false,
  },
  inputSchema: z.object({ resourceId: z.string() }),
  handler: async ({ resourceId }) => ({
    content: [{ type: 'text' as const, text: `Usunięto zasób "${resourceId}". Operacja nieodwracalna.` }],
  }),
});

// Wersja bezpieczna — archiwizacja zamiast usuwania
export const opsArchiveResourceMock = defineTool({
  name: 'ops__archive_resource_mock',
  description: 'Archiwizuje zasób (można przywrócić). Operacja odwracalna i idempotentna.',
  annotations: {
    destructiveHint: false,
    idempotentHint:  true,
    readOnlyHint:    false,
  },
  inputSchema: z.object({ resourceId: z.string() }),
  handler: async ({ resourceId }) => ({
    content: [{ type: 'text' as const, text: `Zarchiwizowano zasób "${resourceId}". Możesz go przywrócić używając ops__restore_resource_mock.` }],
  }),
});
```

**Obserwuj:** Czy klient MCP wyświetla ostrzeżenie przed `ops__delete_resource_mock`? Czy agent woli `archive` zamiast `delete` gdy oba są dostępne?

---

## 12. Kontekstowe zasoby (Resource Links)

**Nazwa:** `catalog__list_items_mock`

**Czego uczy:** Narzędzia mogą zwracać nie tylko tekst, ale też referencje do zasobów MCP (`resource` content type), które klient może odczytać przez mechanizm Resources. Testujesz, czy agent rozumie różnicę między bezpośrednim tekstem a referencją do zasobu wymagającą osobnego odczytu.

**Logika mocka:**

Lista elementów katalogu — każdy element to link do zasobu MCP (`resource` URI), a nie pełna treść. Agent musi zdecydować, które zasoby odczytać przez `resources/read`.

```typescript
export const catalogListItemsMock = defineTool({
  name: 'catalog__list_items_mock',
  description: 'Zwraca listę elementów katalogu jako referencje do zasobów MCP.',
  annotations: { readOnlyHint: true },
  inputSchema: z.object({
    category: z.string().optional().describe('Filtr kategorii'),
  }),
  handler: async ({ category = 'all' }) => {
    const items = [
      { id: 'item_001', title: 'Widget Alpha', uri: 'mock://catalog/item_001' },
      { id: 'item_002', title: 'Widget Beta',  uri: 'mock://catalog/item_002' },
      { id: 'item_003', title: 'Widget Gamma', uri: 'mock://catalog/item_003' },
    ];

    return {
      content: [
        {
          type: 'text' as const,
          text: `Znaleziono ${items.length} elementów w kategorii "${category}". Użyj URI zasobu, aby pobrać szczegóły.`,
        },
        // Referencje do zasobów MCP — agent może je odczytać przez resources/read
        ...items.map(item => ({
          type: 'resource' as const,
          resource: {
            uri:      item.uri,
            mimeType: 'application/json',
            text:     JSON.stringify({ id: item.id, title: item.title, detailsAvailable: true }),
          },
        })),
      ],
    };
  },
});
```

**Obserwuj:** Czy agent rozumie, że `resource` URI wymaga osobnego odczytu? Czy selektywnie pobiera tylko potrzebne zasoby, czy próbuje pobrać wszystkie naraz?

---

## Podsumowanie: co testuje każde narzędzie

| Narzędzie | Testowany wzorzec | Kluczowe pytanie |
|-----------|-------------------|------------------|
| `api__get_records_mock` | Paginacja | Czy agent nawiguje stronami? |
| `fs__write_safe_mock` | Dry run + checksum | Czy agent wykonuje flow dwuetapowy? |
| `fs__read_lines_mock` | Korekta zakresu | Czy agent rozumie koniec pliku? |
| `fs__manage_mock` | Konsolidacja akcji | Czy agent korzysta z jednego narzędzia? |
| `api__trigger_job_mock` | Polling + rate limit | Czy agent czeka, czy się zapętla? |
| `data__get_report_mock` | Structured content | Czy agent używa JSON czy parsujesz tekst? |
| `search__query_mock` | Schema enforcement | Czy agent koryguje parametry po błędzie? |
| `action__confirm_mock` | Elicitation | Czy agent pauzuje dla potwierdzenia? |
| `debug__error_types_mock` | LLM-friendly errors | Czy agent autokoryguje po błędzie? |
| `docs__get_article_mock` | Progressive disclosure | Czy agent czyta nagłówki przed treścią? |
| `ops__delete_resource_mock` | Annotations/hints | Czy klient respektuje destructiveHint? |
| `catalog__list_items_mock` | Resource links | Czy agent selektywnie czyta zasoby? |

---

## Jak zarejestrować wszystkie narzędzia

W `src/shared/tools/registry.ts` dodaj importy i uzupełnij tablicę narzędzi:

```typescript
import { apiGetRecordsMock }    from './api-get-records-mock.js';
import { fsWriteSafeMock }      from './fs-write-safe-mock.js';
import { fsReadLinesMock }      from './fs-read-lines-mock.js';
import { fsManageMock }         from './fs-manage-mock.js';
import { apiTriggerJobMock,
         apiCheckJobMock }      from './api-trigger-job-mock.js';
import { dataGetReportMock }    from './data-get-report-mock.js';
import { searchQueryMock }      from './search-query-mock.js';
import { actionConfirmMock }    from './action-confirm-mock.js';
import { debugErrorTypesMock }  from './debug-error-types-mock.js';
import { docsGetArticleMock }   from './docs-get-article-mock.js';
import { opsDeleteResourceMock,
         opsArchiveResourceMock } from './ops-resource-mock.js';
import { catalogListItemsMock } from './catalog-list-items-mock.js';

export const tools = [
  asRegisteredTool(apiGetRecordsMock),
  asRegisteredTool(fsWriteSafeMock),
  // ...itd.
];
```

Następnie uruchom `bun run mcp:inspector` i obserwuj zachowanie agenta dla każdego scenariusza.

---

*Poprzedni dokument: [07-pomysly-na-serwery-mcp.md](./07-pomysly-na-serwery-mcp.md)*
