---
name: deploy-verify
description: "Deploy + weryfikacja live (Vercel, Cloudflare Pages/Workers). Use after committing/pushing code that auto-deploys, before saying „gotowe/live”, or when the user mentions deploy, Vercel, Cloudflare, build, wdrożenie, „czy poszło na produkcję”. Wymusza weryfikację że zmiana FIZYCZNIE działa na prodzie, nie tylko lokalnie."
---

# Deploy i weryfikacja live

„Wypchnąłem" ≠ „działa na prodzie". Build może paść, cache podać starą wersję, env brakować. Nie mów „live" bez dowodu.

## Kiedy używać / kiedy NIE
- **UŻYJ:** po push z auto-deployem; przed ogłoszeniem „gotowe/live"; debug „padło po deployu".
- **NIE używaj:** czysto lokalne zmiany bez deployu.

## Procedura
1. Po `git push` → **poczekaj na build** (Vercel/CF). Nie ogłaszaj sukcesu z samego push.
2. Sprawdź status builda: `vercel ls` / dashboard / logi. Build FAILED → czytaj log, napraw, nie „powinno przejść".
3. Zweryfikuj URL prod: `curl -L <url>` (Cloudflare Pages i Workers Static Assets serwują bez `.html` → 307, używaj `-L`). Sprawdź że nowa treść JEST (grep charakterystycznego fragmentu zmiany).
3a. **Podbicie `next` (łatka CVE, upgrade):** przed lokalnym buildem `npm ci` / `pnpm i --frozen-lockfile` (`node_modules` potrafi stać na starszej wersji niż lockfile, wtedy testujesz nie to). Po deployu znajdź w logu builda Vercela linię `Detected Next.js version` i porównaj z lockfile. Sam commit `package.json` nie dowodzi, że prod stoi na łatce.
4. Front → **bump cache**: `CACHE_VERSION` w SW + `?v=N` przy `<script>`; inaczej użytkownicy widzą starą wersję.
5. Env vars istnieją na prodzie (nie tylko lokalnie): `vercel env ls`.
6. Zmiana za loginem/nietestowalna curl'em → zweryfikuj podłączenie (callsite/route istnieje) i daj testującemu konkret „kliknij X, oczekuj Y".

## Anti-pattern → pattern
- ❌ „Wypchnąłem, gotowe." → ✅ „Push → build Ready (34s) → curl prod pokazuje nową treść → live."
- ❌ Fix frontu bez bump cache. → ✅ bump SW `?v=` w tym samym commicie.
- ❌ `curl url` bez `-L` na Cloudflare → widzisz 307 nie treść. → ✅ `curl -L`.
- ❌ „Obrazki nie ładują się po deployu, szukam w kodzie." → ✅ 402 z `/_next/image` albo funkcja, która nie wstaje, na Vercel Hobby = wyczerpany limit planu (5K transformacji/mies.; po limicie funkcji stoisz do 30 dni). Najpierw Usage w dashboardzie.
- ❌ og:image sprawdzone w przeglądarce. → ✅ `curl -A "facebookexternalhit/1.1" -L <url>` i sprawdź, że `og:image` siedzi w `<head>` (streaming metadata bez UA bota potrafi wstawić tagi do `<body>`).
- ❌ Zakładasz że env jest na prodzie. → ✅ `vercel env ls` potwierdza.
- ❌ `vercel link` na istniejącym projekcie. → ✅ uwaga: potrafi utworzyć NOWY projekt; sprawdź nazwę.

## Checklist końcowy
- [ ] Build przeszedł (nie tylko push) - sprawdzony status/logi.
- [ ] Prod URL zwraca NOWĄ treść (`curl -L` + grep zmiany).
- [ ] Front: cache zbumpowany; env na prodzie istnieje.
- [ ] Nietestowalne curl'em: podłączenie zweryfikowane + instrukcja dla testującego.
