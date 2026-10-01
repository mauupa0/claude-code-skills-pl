# Next.js 16 i Next na Cloudflare - detale do stack-pitfalls

Czytaj przy upgradzie Next, pisaniu `proxy.ts`, Server Actions, cache albo `next/image`, oraz przy deployu Next na Cloudflare Workers. Krótkie reguły bezpieczeństwa (wersje CVE, proxy to nie auth, `"use cache"`) są w SKILL.md.

## Supabase SSR w `proxy.ts`
- `getClaims()` wołaj ZARAZ po `createServerClient`, zero kodu pomiędzy (to odświeża token).
- Zwracaj `supabaseResponse` zbudowany przez ostatni `setAll`. Inna odpowiedź = losowe wylogowania.
- W Server Components `setAll` w `try/catch` (tam nie da się ustawić nagłówków).
- `getSession()` w kodzie serwera nigdy: czyta niezweryfikowane cookie.
- `getClaims()` weryfikuje lokalnie tylko przy asymetrycznym kluczu podpisu (ES256/RSA). Przy HS256 robi call do Auth jak `getUser()`. Do akcji z kasą zostań przy `getUser()`.

## `proxy.ts` (dawne middleware)
- Plik eksportuje JEDNĄ funkcję `proxy`. next-intl i odświeżanie sesji Supabase składaj w tej jednej funkcji.
- `matcher` musi być stałą (literał), nie wyliczaną wartością.
- Bez `matcher` proxy leci na każdy request, także `_next/static`, `_next/image` i `public/`.
- Migracja z `middleware.ts`: `npx @next/codemod@canary middleware-to-proxy .`. Opcja `runtime` w proxy rzuca błąd (proxy = Node).

## Cache w Next 16
- Po zapisie w Server Action `updateTag(tag)` (read-your-writes).
- `revalidateTag(tag, 'max')` zawsze z drugim argumentem. Forma z jednym jest deprecated: grep po repo przy upgradzie.
- `cacheComponents: true` wywala build, jeśli w route handlerach zostały eksporty `revalidate`, `dynamic`, `dynamicParams`, `fetchCache` albo `runtime`. Grep przed włączeniem.

## `next/image` w 16 (zmiany po cichu)
- `images.qualities` domyślnie `[75]`: `quality={90}` bez 90 w configu daje 75.
- `priority` deprecated. Na JEDEN obraz LCP daj `fetchPriority="high"`. Next domyślnie ładuje lazy, a lazy na obrazie LCP psuje wynik.
- `minimumCacheTTL` domyślnie 4 h zamiast 60 s.
- Każda szerokość, jakość i format = osobna płatna transformacja: `deviceSizes`/`imageSizes` do 3-4 wartości. AVIF tylko przy dużej galerii (osobny cache), domyślnie WebP.
- Obrazy z `public/` importuj statycznie (hash w nazwie + `immutable`).

## Next na Cloudflare Workers (OpenNext)
- `proxy.ts` (Node middleware) tylko eksperymentalnie, wymaga `nodejs_compat`. Build pada, gdy w `node_modules` jest `@opentelemetry/api` (np. z Sentry). Alternatywa: bez proxy, auth w route handlerach i Server Actions.
- `next/image` tylko przez binding CF Images: $0.50/1K transformacji, ok. 10x drożej niż Vercel. Origin ogranicz do własnego bucketu R2.
- Build w GitHub Actions albo na Linuksie, nie na Windowsie.
- Workers Free (10 ms CPU na request) nie uciągnie SSR, realnie plan Paid $5.
- Limit rozmiaru workera: docs CF (64 MiB) i OpenNext (3/10 MiB po kompresji) się kłócą. Rozstrzyga `wrangler deploy --dry-run` na realnym buildzie.
