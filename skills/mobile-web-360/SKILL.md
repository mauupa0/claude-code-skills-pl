---
name: mobile-web-360
description: "Poprawność mobilna web (360px, iOS, PWA). Use when building or reviewing any responsive web UI, a mobile view, a PWA, or when the user mentions mobile, telefon, 360px, overflow, „jeździ w bok”, safe-area, klawiatura zasłania, iOS zoom, dvh. Zawiera pułapki mobilne sprawdzone na produkcji."
---

# Mobile web - 360px i pułapki iOS

Telefon to domyślny ekran większości userów. Testuj realnie na 360px, nie „powinno działać".

## Kiedy używać / kiedy NIE
- **UŻYJ:** responsywne UI, widoki mobilne, PWA, bug „na telefonie inaczej".
- **NIE używaj:** narzędzia czysto desktopowe/CLI.

## Zasady żelazne
1. **Poziomy scroll = bug.** `overflow-x: clip` na `html` I `body`. Znajdź winowajcę: element szerszy niż viewport (sztywna szerokość, `100vw` z paddingiem, długi tekst bez `word-break`).
2. **iOS zoomuje input przy `font-size < 16px`** → `font-size: 16px !important` na `input/select/textarea`.
3. **Wysokości: `dvh` nie `vh`** (pasek adresu iOS zmienia `vh`). `min-height: 100dvh`.
4. **Safe-area:** `viewport-fit=cover` w meta + `padding: env(safe-area-inset-*)` na sticky header/footer (notch, pasek gestów).
5. **Zachowanie UI (hover, tap-targety) steruj przez `pointer: coarse`**, nie szerokość ekranu. `coarse` = dotyk (też iPad), nie „mały ekran". **Rozmiar assetów** (wideo 720p vs 1080p, ciężkie obrazy) dobieraj po krótszym boku ekranu (≤ 600 CSS px = telefon) albo `clientWidth × devicePixelRatio`, nigdy po pointer/UA: iPadOS podaje coarse pointer i UA Maca, więc dostaje rozmazaną wersję telefonową.
6. **Klawiatura zasłania input** → scroll-into-view aktywnego pola; unikaj `position:fixed` na formularzach.
7. **Tap-targety ≥ 44px**; nie polegaj na hover (na telefonie nie ma).
8. **Wideo na iOS:** muted `<video>`, którego nigdy nie odtworzono, nie maluje klatki po seeku → trzymaj poster, aż klip namaluje klatkę; nie usuwaj `playsinline`/`muted`. **Low Power Mode** odrzuca nawet muted `play()` i scrub `currentTime` → `.catch()` na pierwszym `play()` (autoplay z treścią, wideo sterowane scrollem) przełącza stronę na statyczne obrazy.

## PWA na iOS i realny telefon
1. **Web push na iOS (16.4+) działa TYLKO w web appie z ekranu głównego**, nigdy w karcie Safari. `Notification.requestPermission()` wołaj wyłącznie w handlerze kliknięcia (nigdy na load). Userom iOS pokaż ekran-instrukcję „Udostępnij → Dodaj do ekranu początkowego". Push idzie przez APNs bez konta Apple Developer, ale serwer wysyłki (i CSP) musi przepuszczać `*.push.apple.com`.
2. **UE:** web appy na iOS działają, instalacja PWA tylko przez WebKit (nie Chrome/Firefox swoim silnikiem). Statusu web push na iOS w UE w 2026 nikt oficjalnie nie potwierdził → nie raportuj „push na iPhonie działa" bez testu na realnym iPhonie.
3. **Storage na iOS nie jest trwały:** localStorage i IndexedDB wylatują przy LRU i przez ITP, gdy user długo nie wchodzi w interakcję. Sesja Supabase i zgody z localStorage mogą zniknąć → brak sesji to normalny stan (redirect do logowania, baner zgody od nowa), nie crash ani pusty ekran. `navigator.storage.persist()` wołaj, ale Safari przyznaje go heurystycznie.
4. **Od Safari 26 każda strona dodana do ekranu otwiera się jako web app, nawet bez manifestu.** Każda strona (też landing klienta) dostaje manifest: `name`, `short_name`, `start_url`, `display`, `theme_color`, ikony PNG 192 i 512 (to zarazem kryteria instalacji Chrome); `description` + `screenshots` dają duży dialog instalacji na Androidzie. Manifest i SW pisz ręcznie, bez next-pwa/Workbox.
5. **Klawiatura, instalacja PWA, push, powrót z tła: Playwright 360 px tego nie pokaże** → realny Android przez adb. iPhone bez Maca = brak automatyzacji, tylko ręczny test na iPhonie z instrukcją „kliknij X, oczekuj Y".

## Anti-pattern → pattern
- ❌ `height: 100vh` na kontenerze full-screen. → ✅ `100dvh`.
- ❌ `font-size: 14px` na input. → ✅ `16px !important` (inaczej iOS zoom).
- ❌ Test tylko na desktopie „zwężonym". → ✅ realny 360px (DevTools device / headless z prawdziwą szerokością - uwaga: msedge headless `--window-size=360` daje realnie ~476px, mierz w iframe o szerokości 360 px).
- ❌ `@media (hover)` steruje widocznością akcji. → ✅ na `pointer:coarse` akcje stale widoczne.
- ❌ Handler `resize` przelicza layout przy każdej zmianie → strona skacze przy scrollu (chowanie paska adresu zmienia samą wysokość). → ✅ Na dotyku reaguj tylko na zmianę SZEROKOŚCI, obrót obsłuż przez `orientationchange`.

## Checklist końcowy
- [ ] 360px: brak poziomego scrolla; tap-targety klikalne.
- [ ] Inputy `≥16px`; wysokości w `dvh`; safe-area na sticky.
- [ ] Sprawdzone na realnej szerokości (nie zwężony desktop).
- [ ] Push / instalacja PWA / klawiatura: realny Android (adb) albo ręczny test na iPhonie, nie sam Playwright.
