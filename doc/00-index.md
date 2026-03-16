# Dokumentacja edukacyjna: MCP Server Template

> **Cel dokumentacji:** Nauczyć się architektury, wzorców projektowych i decyzji inżynierskich zastosowanych w tym szablonie — czytając kod i rozumiejąc *dlaczego*, nie tylko *co*.
>
> **Adresat:** Osoba znająca dobrze TypeScript, ale nieznająca jeszcze MCP, OAuth 2.1 ani zaawansowanych wzorców architektonicznych po stronie serwera.

---

## Mapa dokumentacji

| Plik | Co zawiera | Czas czytania |
|------|-----------|---------------|
| [01-co-to-jest-mcp.md](./01-co-to-jest-mcp.md) | Protokół MCP od podstaw — JSON-RPC, problem N×M, capabilities | ~15 min |
| [02-architektura-dualna.md](./02-architektura-dualna.md) | Wzorzec adaptera, dual-runtime, podział kodu shared/adapter | ~20 min |
| [03-oauth-i-podwojny-token.md](./03-oauth-i-podwojny-token.md) | OAuth 2.1, PKCE, wzorzec podwójnego tokenu RS→Provider | ~25 min |
| [04-wzorce-projektowe.md](./04-wzorce-projektowe.md) | Strategy, Registry, Factory, Singleton, AsyncLocalStorage | ~30 min |
| [05-warstwa-storage.md](./05-warstwa-storage.md) | Interface pattern, trzy implementacje, AES-256-GCM | ~15 min |
| [06-jak-dodac-tool.md](./06-jak-dodac-tool.md) | Praktyczny przewodnik — od zera do działającego narzędzia | ~20 min |
| [07-pomysly-na-serwery-mcp.md](./07-pomysly-na-serwery-mcp.md) | Katalog integracji: 20+ pomysłów na serwery MCP z różnymi API | ~30 min |
| [08-narzedzia-edukacyjne-mock.md](./08-narzedzia-edukacyjne-mock.md) | 12 narzędzi-mocków bez zewnętrznych usług: paginacja, dry run, polling, elicitation i inne wzorce MCP | ~25 min |

---

## Sugerowana kolejność

Jeśli zaczynasz od zera:

```
01 → 02 → 03 → 04 → 05 → 06
```

Jeśli już rozumiesz MCP i OAuth, zacznij od `04` (wzorce projektowe).

Jeśli chcesz od razu coś zbudować, przeczytaj `01` i `06`.

Jeśli szukasz pomysłów co zbudować i jakich API użyć, przejdź do `07`.

Jeśli chcesz testować zachowanie agenta bez zewnętrznych usług, przejdź do `08`.

---

## Kluczowe pytania, na które odpowiada ta dokumentacja

1. **Co to jest MCP i po co mi serwer?** → `01`
2. **Dlaczego ten sam kod działa na Node.js i Cloudflare Workers?** → `02`
3. **Dlaczego OAuth jest taki skomplikowany i po co ten drugi token?** → `03`
4. **Jakie wzorce projektowe są tu zastosowane i dlaczego?** → `04`
5. **Jak działa warstawa przechowywania danych?** → `05`
6. **Jak napisać własne narzędzie krok po kroku?** → `06`
