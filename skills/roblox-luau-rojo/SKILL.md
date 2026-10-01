---
name: roblox-luau-rojo
description: "Roblox/Luau z Rojo (kod w plikach, Studio jako podgląd). Use when working on a Roblox game, starting a new Roblox game (nowa gra, pomysł na grę, GDD), Luau scripts, a Rojo project (default.project.json), UI or shop in a Roblox game, or when the user mentions Roblox, Luau, Rojo, Studio, ModuleScript, DataStore, parity test, balans gry. Utrzymuje workflow Rojo sync, testy bez Studio i logikę server-authoritative."
---

# Roblox / Luau / Rojo

Gra na Rojo: dysk = źródło prawdy, Rojo auto-syncuje do Studio.

**Najpierw:** `CLAUDE.md` gry i jej testy. Nowa gra od zera: szablon `roblox-rojo-game-template` (Rojo + Wally + testy bez Studio) albo własny `default.project.json` + `rokit.toml`.

## Kiedy używać / kiedy NIE
- **UŻYJ:** kod Luau, projekt Rojo, balans/ekonomia gry, DataStore.
- **NIE używaj:** projekty nie-Roblox.

## Zasady żelazne
1. **Narzędzia:** jest `rokit.toml` w rootcie każdej gry → `rokit install`, potem `rojo`, `wally`, `selene`, `stylua`, `lune` w wersjach z pliku (binarki w `~/.rokit/bin`). Nowa gra bez `rokit.toml`: shim `rojo` bez manifestu odmawia startu, więc najpierw `rokit add rojo-rbx/rojo@7.7.0` w jej rootcie (pierwszy raz na maszynie `rokit trust rojo-rbx/rojo`). Nie instaluj Rojo wingetem (przykrywa shim rokita).
2. **Rojo sync:** `cd <projekt> && rojo serve`, w Studio Plugins→Rojo→Connect. Dysk → Studio bajt-wiernie (naprawia też mojibake).
3. **Przy działającym `rojo serve` kod zmieniasz TYLKO w pliku `.luau` na dysku.** Zero `multi_edit` / `Source =` przez MCP na skryptach zarządzanych przez Rojo: następny sync z dysku nadpisze zmianę. Script Sync Studia NIE włączaj równolegle z Rojo (dwa mechanizmy na tych samych skryptach = konflikty); ma sens tylko przy Team Create z kimś spoza Rojo.
4. **Check bez Studio:** `selene <plik|src>` (lint, z rokit, działa wszędzie). Sama składnia: `luau-compile --null <plik>` (binarka z wydań luau-lang/luau na GitHubie).
5. **Silnik z testem parity (golden) = RNG wstrzykiwany z seedem.** NIE dodawaj `math.random()` ani innych wywołań RNG w środku `simulate()`, złamiesz parity. RNG przychodzi z zewnątrz (np. xorshift32 z seedem). Test: porównaj `RNG(seed)×N` i wynik `simulate(...)` z oracle (np. port w innym języku). Dotyczy gry, która ma taki test.
6. **require cache w Studio:** po zmianie modułu przez Rojo `require` zwraca STARĄ wersję → testuj przez `loadstring(Module.Source)()` fresh.
7. **`execute_luau` (MCP Studio) poza Play działa w EDIT mode** (nie testuje `OnServerInvoke`/DataStore live) → logikę serwerową testuj `loadstring`+mock; gameplay w Play mode (`start_stop_play`). W Play wybierasz datamodel `Server` albo `Client` (każdy ma osobny cache `require`, Edit jest wtedy niedostępny). Test sterowania gracza = `user_keyboard_input` / `user_mouse_input` w Play; `character_navigation` omija realny input, więc nie dowodzi, że sterowanie działa. `execute_luau` i auto-akceptacja skryptów zmieniają OTWARTY place: pracuj po commicie albo na kopii, nigdy auto-accept na placu produkcyjnym.
8. **Server-authoritative:** losowanie/ekonomia/anti-cheat po stronie serwera, klient tylko wyświetla.

## Anti-pattern → pattern
- ❌ Dodajesz `math.random()` do `simulate()` w silniku z testem parity. → ✅ RNG wstrzykiwany z seedem; parity zachowane.
- ❌ Przegląd całego kodu gry przez `script_search` / `script_grep` MCP (ucinają wyniki na 25-50 trafieniach). → ✅ Grep po plikach Rojo w repo; MCP tylko do stanu żywego placu (`search_game_tree`, `inspect_instance`, `execute_luau`).
- ❌ Testujesz zmieniony moduł przez `require` (stara wersja). → ✅ `loadstring(X.Source)()` fresh.
- ❌ Balans „na oko" w Studio. → ✅ symulacja headless (skrypt `lune`, oracle z seedem) + grid-search.
- ❌ Klient liczy dropy/ekonomię. → ✅ serwer authoritative, klient wyświetla.
- ❌ Instalujesz rojo wingetem w grze z `rokit.toml`. → ✅ `rokit install` (wersje przypięte w repo).

## Checklist końcowy
- [ ] Gra z testem parity: zmiana silnika nie dodała RNG do `simulate` (parity trzyma).
- [ ] Kod zmieniony w plikach `.luau` na dysku, nie przez `multi_edit` w Studio.
- [ ] `selene` / `luau-compile --null` przeszły na zmienionych plikach.
- [ ] Logika serwerowa testowana loadstring+mock; ekonomia server-authoritative.
