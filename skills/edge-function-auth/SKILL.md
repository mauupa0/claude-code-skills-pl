---
name: edge-function-auth
description: "Autoryzacja i bezpieczeństwo Supabase Edge Functions (Deno). Use when writing, reviewing, or deploying a Supabase edge function, any serverless HTTP endpoint, or a function using SERVICE_ROLE_KEY; when a function is called by a DB trigger (net.http_post) or by the browser; or when the user mentions edge function, verify_jwt, endpoint bez auth, webhook, service_role. Zawiera anti-patterny typowych dziur (endpointy bez autoryzacji → spam/phishing/wstrzykiwanie)."
---

# Edge Functions - autoryzacja i twardnienie

Edge function bez auth = publiczny endpoint na service_role. Każdy zna URL (widać w Network w przeglądarce) i może POST-ować dowolne body. Typowy scenariusz: funkcja wysyłająca maile bez żadnej weryfikacji to gotowe narzędzie do phishingu z domeny produktu, a funkcja zapisująca dane po id z body pozwala pisać do cudzych rekordów.

## Kiedy używać / kiedy NIE
- **UŻYJ:** każda edge function; endpoint na service_role; funkcja wołana z triggera DB lub z przeglądarki; webhook.
- **NIE używaj:** czysto lokalne skrypty node bez wystawienia na sieć.

## Zasady żelazne
1. **Każdy endpoint autoryzuje wywołującego NA WEJŚCIU.** Dwa wzorce zależnie od kto woła:
   - **Woła trigger DB (net.http_post)** → shared-secret w nagłówku: funkcja sprawdza `req.headers.get('x-hook-secret') === Deno.env.get('HOOK_SECRET')`, inaczej 401. Trigger dokleja nagłówek.
   - **Woła przeglądarka (user)** → `supabase.auth.getUser(jwt)` + **sprawdź własność zasobu** (czy caller jest właścicielem `order_id`/`owner_id` z body). Sam zalogowany ≠ uprawniony do TEGO zasobu.
2. **Nie ufaj żadnemu polu z body.** id, kwoty, emaile z body są sterowalne. Kwotę/uprawnienia licz/weryfikuj serwerowo z DB po id, nie bierz z body.
3. **Idempotencja:** webhook/trigger przyjdzie 2× (retry). Operacja musi być idempotentna (unique constraint jako atomowy claim, guard stanu) - inaczej podwójne naliczenie.
4. **Błąd nie może przepuszczać skutku ubocznego.** Jeśli zapis padł (`console.error`), NIE leć dalej do wysyłki maila/przelewu. Fail-closed.
5. **Sekrety tylko z `Deno.env`**, nigdy w kodzie/logach. Nie loguj pełnych bodies z danymi osobowymi.
6. **Nowe klucze API (`sb_publishable_`/`sb_secret_`) to NIE JWT.** Legacy `anon`/`service_role` wygasają do końca 2026. Nowe klucze idą w nagłówku `apikey`, nie w `Authorization: Bearer`, a sam `verify_jwt` nie uwierzytelnia wywołania samym kluczem.
   - `verify_jwt=true` to nie autoryzacja: przepuszcza każdego z publicznym kluczem. Auth zawsze w kodzie (pkt 1).
   - Trigger/cron z wklejonym w SQL `Bearer <anon JWT>` padnie po wyłączeniu legacy → przepnij na shared-secret.
   - Kolejność: auth w kodzie → test na DEV → `verify_jwt=false` → dopiero wyłączenie legacy kluczy.
   - `sb_secret_` z przeglądarki dostaje 401 po `User-Agent`: heurystyka, nie ochrona. Sekret tylko na serwerze.
7. **`getUser()` vs `getClaims()`.** `getClaims()` weryfikuje JWT lokalnie (JWKS) tylko przy asymetrycznym kluczu podpisu (ES256/RS256); przy HS256 i tak woła Auth. Nie zakładaj, że widzi ban/wylogowanie przed wygaśnięciem tokenu: kasa, wypłaty, uprawnienia → `getUser()`; zwykłe odczyty → `getClaims()`.
   - Rotacja kluczy podpisu: funkcje z `verify_jwt` mogą się wysypać (wyłącz, weryfikuj `getClaims()` w kodzie); własna weryfikacja na legacy JWT secret (`jose`, `jsonwebtoken`, worker CF) padnie. Revoke starego klucza najwcześniej po 1 h 15 min + czas życia tokenu (szybciej = wylogowanie wszystkich).

## Anti-pattern → pattern
- ❌ `serve()` → parse body → klient na SERVICE_ROLE, zero auth. → ✅ Pierwsze linie: shared-secret ALBO `getUser()`+ownership; dopiero potem logika.
- ❌ `send-email` publiczny, subject/treść z body → brandowany phishing na dowolny adres. → ✅ Shared-secret na wejściu + (opcjonalnie) rate-limit per adres.
- ❌ Upsert padł → `console.error` → kod leci do `sendMail(body.email)`. → ✅ `if (error) return 400` PRZED wysyłką; mail tylko po udanym zapisie.
- ❌ `refund/charge` ufa `kwota` z body. → ✅ Kwotę czyta funkcja z DB po `order_id`; body podaje tylko id.
- ❌ Webhook robi `update(status)` bez guardu → retry cofa stan. → ✅ `.in('status', [dozwolone_stany])` - retry na późniejszym stanie = no-op.

## Przykład (few-shot)
Wejście: „funkcja `send-email` wysyła maila po zdarzeniu w DB".
Oczekiwane: na wejściu `if (req.headers.get('x-hook-secret') !== Deno.env.get('HOOK_SECRET')) return new Response('unauthorized',{status:401})`; trigger `net.http_post` dokleja nagłówek `x-hook-secret`; treść walidowana; brak rate-limitu → dodać per-adres. Zero ścieżki gdzie body samo decyduje o odbiorcy bez sekretu.

## Checklist końcowy
- [ ] Pierwsze instrukcje funkcji = autoryzacja (shared-secret dla triggerów / getUser+ownership dla usera).
- [ ] Żadne pole z body nie decyduje o kwocie/uprawnieniu bez weryfikacji z DB.
- [ ] Idempotencja: retry webhooka/triggera = no-op (unique/guard stanu).
- [ ] Błąd zapisu → fail-closed (brak maila/przelewu po nieudanym zapisie).
- [ ] Sekrety z env, nie w kodzie/logach; brak logowania pełnych danych osobowych.
- [ ] Zweryfikowane curlem: bez tokenu/sekretu → 401 ORAZ z samym `apikey: sb_publishable_...` (bez Bearer usera i bez sekretu) → 401, nie 200.
