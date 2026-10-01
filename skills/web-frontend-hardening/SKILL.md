---
name: web-frontend-hardening
description: "Twardnienie frontendu web (vanilla JS/HTML + Supabase realtime, PWA). Use when writing or reviewing front-end code that renders user/DB data, uses inline onclick handlers, Supabase realtime subscriptions, Service Worker caching, or shows money/critical data; or builds public forms (contact, brief, signup), Turnstile/captcha, rate limiting, analytics/GA4 and cookie consent; or when the user mentions XSS, escHtml, realtime, Service Worker, cache, „po F5 puste”, cichy fail, formularz, spam, boty, captcha, cookies, zgoda, baner. Zawiera anti-patterny typowych bugów (escHtml w onclick, realtime bez recovery, saldo=0 przy błędzie)."
---

# Frontend web - twardnienie

Panele w vanilla JS + Supabase realtime + PWA renderują dane z DB i pieniądze. Trzy typowe klasy bugów: XSS w inline handlerach, realtime bez recovery, cichy fail na widoku kasy.

## Kiedy używać / kiedy NIE
- **UŻYJ:** render danych z usera/DB; inline `onclick`; realtime subscribe; Service Worker; widoki z saldem/kasą; formularze (też publiczne: captcha, rate limit); analityka i baner cookies.
- **NIE używaj:** czysto statyczny content bez danych dynamicznych.

## Zasady żelazne
1. **XSS w inline handlerze: `escHtml` NIE wystarcza.** Parser HTML dekoduje `&#39;` z powrotem do apostrofu ZANIM JS sparsuje `onclick` → breakout ze stringa. Do wartości w `onclick="fn('${x}')"` używaj `escapeJsArg(x)`, nie `escHtml(x)`. Najlepiej: `addEventListener` + id-lookup zamiast inline.
2. **Realtime musi mieć recovery.** `.subscribe()` bez callbacka statusu = po utracie połączenia panel niesvieży do F5. Dodaj: `.subscribe((status)=>{ if(status==='SUBSCRIBED') refetch(); })` + refetch na `visibilitychange→visible` (throttled). Anon socket: `realtime.setAuth(token)` przed subscribe.
3. **Błąd na widoku kasy NIE może wyglądać jak zero.** `const {data}=await sb.from('wallets')...` bez `error` → przy padzie `data=null` → `saldo=0`. Zawsze destrukturyzuj `error`, przy błędzie renderuj stan błędu z „Ponów", nie `renderWallet(0)`.
4. **Każdy fetch ma error path widoczny userowi.** `.catch(()=>{})` maskuje awarię jako pusty stan. Ustaw flagę błędu → `ErrorState` z retry, nie `EmptyState`.
5. **PWA cache-bust po KAŻDYM fixie frontu:** bump `CACHE_VERSION` w SW + `?v=N` przy `<script src>`. Bez tego użytkownik widzi starą wersję i „dalej nie działa".
6. **Aktualizacja SW:** nowy SW czeka w `waiting`, aż zamkniesz WSZYSTKIE karty ze starym. `skipWaiting()` bez przeładowania = stary HTML sterowany nowym SW (mieszanka wersji). Wzorzec: SW w `waiting` → baner „Nowa wersja, odśwież" → `postMessage({type:'SKIP_WAITING'})` → w SW `self.skipWaiting()` → na stronie `controllerchange` → `location.reload()` raz (flaga przeciw pętli).
7. **`sw.js` serwuj z `Cache-Control: no-cache`** (Vercel `headers` w configu, Cloudflare `_headers`), bo CDN potrafi oddać stary plik. Pliki z `importScripts()` z długim cache HTTP NIE wyzwalają aktualizacji SW → wersjonuj ich URL (`?v=N`) razem z `CACHE_VERSION`.
8. **SW offline fallback:** Cloudflare Pages i Workers Static Assets (`html_handling`) serwują URL bez `.html` (307 redirect). Fallback do `caches.match('/')` nie `'/index.html'` (redirected response = network error w nawigacji).
9. **Nie czyść inputu przed potwierdzeniem wysyłki.** `input.value=''` przed `await insert` → przy błędzie treść przepada. Czyść PO sukcesie; w error path przywróć `input.value=text`.
10. **Pending/localStorage kasuj tylko przy sukcesie/definitywnym błędzie**, nie po każdej próbie - transient błąd sieci nie może kasować danych na zawsze.
11. **Publiczny formularz (kontakt, brief, zgłoszenie, rejestracja): Turnstile działa TYLKO z `siteverify` na serwerze** (`POST https://challenges.cloudflare.com/turnstile/v0/siteverify`), sam widget = zero ochrony. Token ważny 300 s i jednorazowy: po błędzie wysyłki `turnstile.reset()` przed retry, inaczej `timeout-or-duplicate`. Brak sekretu albo błąd weryfikacji = odmowa, nie przepuszczenie. Do tego honeypot (w testach E2E wypełniaj pola po nazwie, nie wszystkie inputy).
12. **Rate limit kluczuj po user id/email + IP, nie samym IP** (kampus, NAT komórkowy = setki ludzi na jednym IP). CF binding `ratelimits` jest per lokalizacja i przybliżony (okno tylko 10 albo 60 s): na spam tak, na limity kasowe i kwotowe nie. Na Vercelu licznik okna w tabeli Supabase + RPC SECURITY DEFINER zamiast Upstash (zero nowych zależności).
13. **Analityka i piksele tylko po zgodzie** (PL: PKE art. 399 od 10.11.2024, obejmuje też localStorage i własne id odwiedzającego): tag GA4/Ads ładuj dopiero po „Akceptuj"; przy Consent Mode v2 tryb Basic (`gtag('consent','default', ...)` z wszystkim `'denied'` przed tagiem), nigdy Advanced (pingi przed zgodą). „Odrzuć" na pierwszej warstwie równie widoczne jak „Akceptuj", zero pre-zaznaczeń, wycofanie zgody linkiem w stopce. Sesja Supabase = niezbędne, bez zgody. Wzorzec: osobny `ga.js` (init tylko przy consent `'all'`) + osobny skrypt banera zgody.

## Anti-pattern → pattern
- ❌ `onclick="openThread('${escHtml(id)}')"`. → ✅ `onclick="openThread('${escapeJsArg(id)}')"` lub `addEventListener`+dataset.
- ❌ `.subscribe()` bez statusu. → ✅ `.subscribe(s=>s==='SUBSCRIBED'&&refetch())` + refetch na focus.
- ❌ `const {data}=await sb.from('wallets')...; renderWallet(data?.saldo||0)`. → ✅ `const {data,error}=...; if(error) return renderError('Nie wczytano salda', retry)`.
- ❌ `fetch(...).catch(()=>{})` → user widzi „brak danych" przy awarii sieci. → ✅ `catch → setError(true)` → ErrorState z onRetry.
- ❌ `input.value=''` przed `await insert`. → ✅ czyść po sukcesie; error path: `input.value=text`.
- ❌ Widget Turnstile na formularzu, token nigdzie nie sprawdzany (albo `catch { return true }`). → ✅ `siteverify` w route handlerze/Edge Function, błąd lub brak sekretu = odmowa.

## Przykład (few-shot)
Wejście: „lista rozmów z przyciskiem otwierającym czat".
Oczekiwane: render przez `escapeJsArg` w inline (lub addEventListener); subscribe z callbackiem statusu + refetch na reconnect/focus; jeśli fetch listy padnie → ErrorState z „Ponów", nie pusty stan; po zmianie tego pliku → bump SW + `?v=`.

## Checklist końcowy
- [ ] Wartości w inline `onclick` przez `escapeJsArg` (nie escHtml) - lub addEventListener.
- [ ] Realtime: callback statusu + refetch na reconnect i visibilitychange; setAuth przed subscribe.
- [ ] Widoki kasy/danych: `error` destrukturyzowany, stan błędu zamiast „0"/pustego.
- [ ] Każdy fetch: widoczny error path z retry (nie `.catch(()=>{})`).
- [ ] Po zmianie frontu: bump CACHE_VERSION + `?v=N`.
- [ ] `sw.js` z `Cache-Control: no-cache`; nowy SW aktywowany przez baner + `SKIP_WAITING` + jednorazowy reload.
- [ ] Publiczny formularz: `siteverify` na serwerze (fail-closed) + honeypot + rate limit po user/email + IP.
- [ ] GA4/piksele ładowane dopiero po zgodzie; „Odrzuć" równorzędne z „Akceptuj".
- [ ] Input czyszczony po sukcesie; pending kasowany tylko przy sukcesie/definitywnym błędzie.
