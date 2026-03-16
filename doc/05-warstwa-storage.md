# 05 — Warstwa przechowywania danych

## 5.1 Dwa rodzaje danych do przechowywania

Serwer MCP przechowuje dwa rodzaje danych:

| Dane | Interfejs | Co przechowuje |
|------|-----------|----------------|
| Tokeny | `TokenStore` | Mapowanie RS token → provider token (access + refresh) |
| Sesje | `SessionStore` | Stan sesji MCP (kto połączony, jaka wersja protokołu) |

Oba interfejsy mają trzy implementacje dostosowane do środowiska.

---

## 5.2 TokenStore — przechowywanie tokenów OAuth

### Struktura danych

```typescript
// Uproszczone — co jest przechowywane dla każdego RS tokenu
interface TokenEntry {
  rsToken: string;              // klucz (Resource Server token)
  providerAccessToken: string;  // token do Google/GitHub/Linear API
  providerRefreshToken?: string;// token do odświeżenia access tokenu
  expiresAt: number;            // timestamp wygaśnięcia (ms)
  scopes: string[];             // przyznane uprawnienia
  apiKey?: string;              // powiązanie z API key klienta
}
```

### Trzy implementacje

**FileTokenStore** (Node.js, `src/shared/storage/file.ts`):
```
Backend: plik JSON na dysku (.data/tokens.json)
Kiedy: Node.js development i production
Zaleta: persists między restartami, działa bez zewnętrznych zależności
Wada: nie działa przy wielu instancjach (race conditions)

Cykl zapisu:
  1. Zmiana w pamięci (Map<string, TokenEntry>)
  2. Co 5 minut lub przy graceful shutdown → flush do pliku JSON
  3. Przy starcie → wczytanie z pliku (z filtrowaniem wygasłych)
```

**MemoryTokenStore** (oba środowiska, `src/shared/storage/memory.ts`):
```
Backend: Map<string, TokenEntry> w pamięci procesu
Kiedy: testy, development bez OAuth, Workers fallback
Zaleta: zero setup, szybki
Wada: dane giną po restarcie (lub po zakończeniu Workers request)
```

**KvTokenStore** (Workers, `src/shared/storage/kv.ts`):
```
Backend: Cloudflare KV (globally replicated key-value store)
Kiedy: Cloudflare Workers production
Zaleta: persists globalnie, automatyczne TTL, bez pliku
Wada: eventual consistency (replikacja ~60s), płatny (powyżej limitu)
```

---

## 5.3 SessionStore — przechowywanie sesji

### Struktura sesji

```typescript
interface SessionData {
  sessionId: string;
  apiKey: string;               // kto jest właścicielem sesji
  initialized: boolean;         // czy klient wysłał 'initialized'
  protocolVersion: string;      // np. "2025-11-25"
  createdAt: number;
  lastAccessedAt: number;
}
```

### Trzy implementacje

**MemorySessionStore** (Node.js default, `src/shared/storage/memory.ts`):
```
Backend: Map + LRU eviction
Limit sesji: 5 per API key (konfigurowalny)
LRU = Least Recently Used — najstarsze nieużywane sesje są usuwane gdy limit osiągnięty
```

**SqliteSessionStore** (Node.js optional):
```
Backend: SQLite przez Drizzle ORM
Kiedy: potrzebujesz persystencji sesji między restartami
Wymaga: bun add drizzle-orm better-sqlite3
```

**KvSessionStore** (Workers, `src/adapters/http-workers/`):
```
Backend: Cloudflare KV z TTL 24h
Kiedy: Workers (wymagany, bo Workers jest stateless między requestami)
```

---

## 5.4 Szyfrowanie AES-256-GCM

Provider tokens to wrażliwe dane. Są szyfrowane przed zapisem do pliku/KV.

```
RS_TOKENS_ENC_KEY=base64url-encoded-32-byte-key
```

**Jak działa AES-256-GCM:**

```
Klucz: 256 bitów (32 bajty, z RS_TOKENS_ENC_KEY)
IV: 96 bitów (12 bajtów, losowy dla każdego szyfrowania!)
AuthTag: 128 bitów (16 bajtów, weryfikuje integralność)

Szyfrowanie:
  input: plaintext (JSON z tokenami)
  output: base64url(IV || ciphertext || authTag)

Odszyfrowanie:
  input: base64url string
  decode → IV, ciphertext, authTag
  AES-GCM decrypt z IV i kluczem
  weryfikacja authTag (jeśli fail → tamper detected → wyjątek)
```

**Dlaczego IV jest losowy dla każdego szyfrowania?** Gdyby IV był stały, identyczne plaintext dałyby identyczny ciphertext. Atakujący mógłby wykryć czy dwa tokeny mają tę samą wartość bez deszyfrowania. Losowy IV każdorazowo zapewnia że to samo plaintext → różny ciphertext.

**Co to jest "256" w AES-256?** Długość klucza. AES-128 = 128-bitowy klucz, AES-256 = 256-bitowy. Dłuższy klucz = więcej możliwych kombinacji = trudniejszy brute force. AES-256 uznawany jest za "quantum-resistant" (bezpieczny nawet wobec komputerów kwantowych w przewidywalnej przyszłości).

---

## 5.5 Generowanie klucza szyfrowania

```bash
# Generuje 32 bajty losowe, konwertuje do base64url (bez paddingu)
openssl rand -base64 32 | tr -d '=' | tr '+/' '-_'

# Przykładowy output:
# Xk3mP9vLqR8nWjY2eZsT5uBhC7dFoA1i4gKxNpQrMwU
```

**Dlaczego base64url, a nie hex?** Base64url jest bezpieczny w URL-ach (nie zawiera `+`, `/`, `=` które mają specjalne znaczenie w URL/shell). 32 bajty = 256 bitów. Base64url zakoduje je w 43 znakach.

---

## 5.6 Inicjalizacja i wzorzec Singleton

```typescript
// src/index.ts (Node.js)
const tokenStore = new FileTokenStore({
  filePath: config.rsTokensFile,
  encryptionKey: config.rsTokensEncKey, // opcjonalne
});
const sessionStore = new MemorySessionStore({ maxSessionsPerKey: 5 });

initializeStorage(tokenStore, sessionStore);
// Od teraz: getTokenStore() i getSessionStore() działają wszędzie
```

```typescript
// src/worker.ts (Workers)
const tokenStore = new KvTokenStore(env.TOKENS, {
  encryptionKey: config.rsTokensEncKey,
});
const sessionStore = new KvSessionStore(env.TOKENS);

initializeStorage(tokenStore, sessionStore);
// Guard w singleton.ts zapobiega podwójnej inicjalizacji
```

---

## 5.7 Cleanup i TTL

Tokeny wygasają. Każda implementacja ma swój mechanizm czyszczenia:

```
FileTokenStore:
  - Przy ładowaniu pliku: pomija wpisy z expiresAt < Date.now()
  - Opcjonalny interval cleanup (np. co godzinę)
  - Graceful shutdown: flush do pliku przed zamknięciem

MemoryTokenStore:
  - Przy dostępie: sprawdza expiresAt → usuwa wygasłe
  - Opcjonalny setInterval dla background cleanup

KvTokenStore:
  - TTL ustawione na poziomie KV (put z expirationTtl)
  - Cloudflare automatycznie usuwa wygasłe klucze
  - Zero kodu cleanup po stronie aplikacji
```

---

## 5.8 Rotacja klucza szyfrowania

Co się dzieje przy zmianie `RS_TOKENS_ENC_KEY`?

```
Stary klucz: A
Nowy klucz: B

Wszystkie tokeny zaszyfrowane kluczem A → nie dają się odszyfrować kluczem B
→ Użytkownicy muszą ponownie przejść OAuth flow
```

Nie ma automatycznej re-szyfrowania (migration). To świadoma decyzja: prostota > convenience. Jeśli potrzebujesz płynnej rotacji, musisz:
1. Deszyfrować starym kluczem
2. Szyfrować nowym kluczem
3. Zapisać nowe wartości
4. Zmienić klucz

---

## Podsumowanie

| Implementacja | Runtime | Backend | Persystencja | Szyfrowanie |
|--------------|---------|---------|-------------|-------------|
| FileTokenStore | Node.js | JSON plik | Tak (dysk) | Opcjonalne (AES-256-GCM) |
| MemoryTokenStore | Oba | RAM | Nie | Nie |
| KvTokenStore | Workers | Cloudflare KV | Tak (globalnie) | Opcjonalne (AES-256-GCM) |
| MemorySessionStore | Node.js | RAM + LRU | Nie | Nie |
| SqliteSessionStore | Node.js | SQLite | Tak (dysk) | Nie |
| KvSessionStore | Workers | Cloudflare KV | Tak (globalnie) | Nie |

**Dalej:** [06 — Jak dodać własne narzędzie (tool)](./06-jak-dodac-tool.md)
