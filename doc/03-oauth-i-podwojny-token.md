# 03 — OAuth 2.1 i wzorzec podwójnego tokenu

## 3.1 Dlaczego OAuth jest trudny?

OAuth to standard delegowania autoryzacji. Zamiast "podaj mi hasło do Gmaila", mówisz "Google, pozwól tej aplikacji czytać moje maile". Brzmi prosto, ale implementacja ma wiele pułapek bezpieczeństwa.

Ten szablon implementuje **OAuth 2.1** — uproszczoną, bardziej bezpieczną wersję OAuth 2.0. Kluczowe uproszczenie: **PKCE jest obowiązkowe** (więcej za chwilę).

---

## 3.2 Trzy strony w OAuth

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Resource Owner         MCP Client           MCP Server │
│  (Użytkownik)           (np. Claude         (ten kod)   │
│                          Desktop)                       │
│       │                      │                   │      │
│       │                      │── /authorize ─────→│     │
│       │                      │                   │      │
│       │←── Przekierowanie ──────────────────────│      │
│       │    do Google login                              │
│       │                                                 │
│       │── Loguje się do Google ──────────────────────→ │
│       │                                        │        │
│       │                 ┌── Google zwraca code─┘        │
│       │                 │                               │
│       │                 └──→ /token (wymiana code)      │
│       │                      │                         │
│       │                      │←── RS token ───────────│
│       │                      │                         │
│       │                      │── /mcp (z RS token) ──→ │
└─────────────────────────────────────────────────────────┘
```

- **Resource Owner** = Ty (użytkownik który ma konto Google)
- **MCP Client** = aplikacja AI (Claude Desktop, IDE, agent)
- **MCP Server** = ten kod — działa jako pośrednik
- **Provider** = Google, GitHub, Linear, Spotify (dostawca API)

---

## 3.3 PKCE — ochrona przed przechwyceniem kodu

PKCE (Proof Key for Code Exchange, wymawia się "pixie") rozwiązuje problem: co jeśli ktoś przechwyci `code` z przekierowania?

**Bez PKCE:**
```
1. Klient prosi o autoryzację
2. Provider zwraca code w URL → ktoś może to przechwycić
3. Atakujący wymienia code na token → kompromitacja konta
```

**Z PKCE:**
```
1. Klient generuje losowy code_verifier (64 bajty)
2. Klient oblicza code_challenge = SHA256(code_verifier) → base64url
3. Klient wysyła code_challenge do /authorize (ale NIE code_verifier!)
4. Provider zapamiętuje challenge, zwraca code
5. Klient wysyła code + code_verifier do /token
6. Serwer sprawdza: SHA256(code_verifier) == code_challenge ?
   → Tak: OK
   → Nie: Odmowa
```

Nawet jeśli atakujący przechwyci `code`, nie ma `code_verifier` — nie może wymienić go na token.

```typescript
// Jak to wygląda w kodzie (src/shared/oauth/flow.ts):
const codeVerifier = generateRandomString(64);        // losowe bajty
const codeChallenge = base64url(sha256(codeVerifier)); // hash

// Do /authorize wysyłamy:
{ code_challenge: codeChallenge, code_challenge_method: "S256" }

// Do /token wysyłamy:
{ code: receivedCode, code_verifier: codeVerifier }
```

---

## 3.4 Wzorzec podwójnego tokenu — kluczowa decyzja architektoniczna

To jest najważniejszy wzorzec bezpieczeństwa w tym szablonie.

**Problem:** Klient MCP potrzebuje autoryzacji do wywołania narzędzi. Narzędzia potrzebują tokenu dostępu do Google API. Jak to połączyć?

**Naiwne rozwiązanie (ZŁE):**
```
Klient → Google token → /mcp
→ Serwer używa Google tokenu bezpośrednio
```
Problemy: klient widzi token Google, klient może bezpośrednio wywołać Google API bez naszej kontroli, trudno odwołać dostęp.

**Wzorzec podwójnego tokenu (DOBRE):**
```
┌──────────────────────────────────────────────────────┐
│                    MCP Server                        │
│                                                      │
│  RS Token (Resource Server Token)                   │
│  ─────────────────────────────────                  │
│  To jest token KLIENTA MCP                          │
│  Klient nigdy nie widzi niczego innego              │
│                │                                     │
│                ↓ Mapowanie (TokenStore)              │
│                                                      │
│  Provider Token (Google/GitHub/Linear access token) │
│  ──────────────────────────────────────────────     │
│  Używany wyłącznie wewnętrznie do API calls         │
│  Nigdy nie opuszcza serwera                         │
└──────────────────────────────────────────────────────┘
```

**Jak to działa:**

```
1. OAuth flow:
   Użytkownik loguje się → Google zwraca access_token + refresh_token
   Serwer generuje losowy RS token (opaque string)
   Serwer zapisuje: RS_TOKEN → { google_access_token, google_refresh_token }
   Serwer zwraca RS token klientowi

2. Każde żądanie MCP:
   Klient: Authorization: Bearer <RS_TOKEN>
   Serwer: lookupToken(RS_TOKEN) → { google_access_token }
   Serwer wywołuje Google API z google_access_token
   Narzędzie działa → serwer zwraca wynik klientowi
```

**Dlaczego to lepsze:**
- Klient MCP nie ma dostępu do Google API z pominięciem serwera
- Możesz unieważnić RS token bez unieważniania Google tokenu
- Google token jest szyfrowany w spoczynku (AES-256-GCM)
- Kiedy Google token wygaśnie, serwer go odświeżyć bez angażowania klienta

---

## 3.5 Serwer jako OAuth Authorization Server

Ten szablon nie tylko *używa* OAuth — on *implementuje* OAuth Authorization Server (AS). To oznacza, że klient MCP wykonuje OAuth flow *wobec tego serwera*, a serwer wykonuje OAuth flow *wobec zewnętrznego dostawcy*.

```
Claude Desktop ──→ OAuth flow ──→ MCP Server (Authorization Server)
                                          │
                                          ↓ OAuth flow
                                   Google/GitHub/Linear
```

MCP Server w trybie OAuth udostępnia:

```
/.well-known/oauth-authorization-server  ← metadane AS (discovery)
/authorize                               ← start flow
/oauth/callback                          ← redirect z Google
/token                                   ← wymiana code → token
/revoke                                  ← unieważnienie tokenu
/register                                ← dynamiczna rejestracja klienta
```

**Co to jest discovery?** Klient MCP nie musi znać wszystkich endpointów z góry. Odczytuje je z `/.well-known/oauth-authorization-server`:

```json
{
  "authorization_endpoint": "https://serwer.com/authorize",
  "token_endpoint": "https://serwer.com/token",
  "code_challenge_methods_supported": ["S256"],
  ...
}
```

---

## 3.6 Odświeżanie tokenów (proactive refresh)

Tokeny dostępu wygasają (typowo po 1h). Wzorzec odświeżania:

**Reaktywne (złe):** Wywołaj API → dostaniesz 401 → odśwież token → spróbuj ponownie.
Problem: request fail widoczny dla użytkownika.

**Proaktywne (dobre):** Przed każdym użyciem sprawdź czas wygaśnięcia tokenu. Jeśli wygasa za mniej niż 60 sekund — odśwież *zanim* użyjesz.

```typescript
// src/shared/oauth/refresh.ts (uproszczony)
async function getValidProviderToken(rsToken: string): Promise<string> {
  const tokenData = await tokenStore.get(rsToken);

  const expiresIn = tokenData.expiresAt - Date.now();
  const BUFFER = 60_000; // 1 minuta buforu

  if (expiresIn < BUFFER) {
    // Odśwież przed użyciem
    const newTokens = await refreshProviderToken(tokenData.refreshToken);
    await tokenStore.set(rsToken, newTokens);
    return newTokens.accessToken;
  }

  return tokenData.accessToken;
}
```

---

## 3.7 CIMD — dynamiczna rejestracja klienta

CIMD (Client ID Metadata Document, zgodnie z SEP-991) to mechanizm, w którym klient MCP *nie musi być wcześniej zarejestrowany* na serwerze. Zamiast tego serwer pobiera metadane klienta z publicznie dostępnego dokumentu.

```
Klient wysyła: client_id = "https://claude.ai/.well-known/mcp-client"
Serwer pobiera: GET https://claude.ai/.well-known/mcp-client
                → { client_name: "Claude", redirect_uris: [...] }
Serwer akceptuje klienta (lub odrzuca, jeśli domena nie jest na allowliście)
```

**SSRF protection** — dlaczego to wymaga ochrony? Bez niej atakujący mógłby podać `client_id = "http://192.168.1.1/admin"` i zmusić serwer do wywołania wewnętrznych adresów IP. Dlatego `src/shared/oauth/ssrf.ts` sprawdza:
- Czy domena jest na allowliście (`CIMD_ALLOWED_DOMAINS`)
- Czy nie jest to adres prywatny (RFC 1918)
- Czy nie przekracza limitu rozmiaru odpowiedzi

---

## 3.8 Szyfrowanie tokenów w spoczynku

RS token → provider token mappings są wrażliwymi danymi. Przechowywane są zaszyfrowane AES-256-GCM:

```typescript
// src/shared/crypto/aes-gcm.ts
// AES-256-GCM to symetryczne szyfrowanie authenticated

// Szyfrowanie:
const iv = randomBytes(12);      // initialization vector, losowy, 96 bitów
const key = importKey(RS_TOKENS_ENC_KEY);
const ciphertext = aesGcmEncrypt(plaintext, key, iv);
const stored = base64(iv + ciphertext + authTag);  // authTag = integrity check

// Odszyfrowanie:
const { iv, ciphertext, authTag } = decode(stored);
const plaintext = aesGcmDecrypt(ciphertext, key, iv, authTag);
// Jeśli ktoś zmodyfikował dane → authTag nie pasuje → wyjątek
```

**Dlaczego GCM?** AES-GCM to *authenticated encryption* — oprócz szyfrowania, zapewnia integralność. Jeśli ktoś zmodyfikuje zaszyfrowane dane, deszyfrowanie się nie powiedzie (authentication tag nie będzie pasował). To zapobiega atakom typu "bit flipping".

---

## 3.9 Strategie autoryzacji bez OAuth

Nie każdy serwis wymaga OAuth. Szybkie podsumowanie pozostałych strategii:

```typescript
// bearer — statyczny token
AUTH_STRATEGY=bearer
BEARER_TOKEN=sk-proj-abc123
→ Każde żądanie: Authorization: Bearer sk-proj-abc123

// api_key — klucz API w nagłówku
AUTH_STRATEGY=api_key
API_KEY=my-api-key
API_KEY_HEADER=x-api-key  // domyślna nazwa nagłówka
→ Każde żądanie: X-Api-Key: my-api-key

// custom — wiele nagłówków
AUTH_STRATEGY=custom
CUSTOM_HEADERS=X-Tenant-ID:acme,X-Region:eu
→ Każde żądanie: X-Tenant-ID: acme, X-Region: eu

// none — brak auth (publiczne API lub dev)
AUTH_STRATEGY=none
```

Wszystkie strategie konwergują do tego samego punktu: `context.providerToken` lub `context.resolvedHeaders` w handlerze narzędzia. Narzędzie nie wie, która strategia jest używana.

---

## Podsumowanie

| Wzorzec | Gdzie w kodzie | Po co |
|---------|---------------|-------|
| OAuth 2.1 + PKCE | `src/shared/oauth/flow.ts` | Bezpieczna delegacja autoryzacji |
| Podwójny token (RS→Provider) | `src/shared/storage/` | Izolacja klienta od credentials dostawcy |
| OAuth Authorization Server | `src/http/auth-app.ts` | Serwer jako pośrednik OAuth |
| Proactive refresh | `src/shared/oauth/refresh.ts` | Zero-downtime odświeżanie tokenów |
| CIMD + SSRF protection | `src/shared/oauth/cimd.ts`, `ssrf.ts` | Bezpieczna dynamiczna rejestracja klientów |
| AES-256-GCM | `src/shared/crypto/aes-gcm.ts` | Szyfrowanie tokenów w spoczynku |

**Dalej:** [04 — Wzorce projektowe w kodzie](./04-wzorce-projektowe.md)
