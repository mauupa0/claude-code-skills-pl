---
name: lokalna-apka-zero-deps
description: "Wzorzec lokalnych apek (Node zero-deps, dane w plikach, offline). Use when building or modifying a local desktop-style app that runs on your own machine, or when the user mentions lokalna apka, start.bat, offline, zero zależności, node:sqlite, dane w plikach, sync na pendrive. Utrzymuje styl: brak zależności runtime, dane w repo, sync przez git."
---

# Lokalne apki zero-deps

Apki uruchamiane lokalnie: mają być przenośne (pendrive/laptop), offline, bez instalacji. Reguła: **zero zależności runtime**.

## Kiedy używać / kiedy NIE
- **UŻYJ:** lokalna apka Node/desktop; `start.bat`; dane w plikach; sync przez git.
- **NIE używaj:** apka webowa na Vercel/Supabase (to inny stack).

## Zasady żelazne
1. **Zero zależności runtime.** Vanilla Node (`http`, `fs`), wbudowany `node:sqlite`. Bez npm install do działania. Frontend: vanilla JS/HTML, fonty self-hosted (woff2 lokalnie, nie googleapis).
2. **Dane w plikach w repo** (MD/JSON), nie w localStorage/chmurze - dzięki temu sync przez git działa i dane przeżywają.
3. **`start.bat`** = jedno kliknięcie: odpala serwer + otwiera przeglądarkę. Node z PATH lub portable `node.exe` w folderze. Na Linuksie i macOS `.bat` nie pójdzie: `node app/server.js` (albo `start.sh` obok `start.bat`, jeśli apka ma działać i na Windows, i na Linuksie).
4. **SQLite:** `node:sqlite`; `LOWER()` działa tylko na ASCII → normalizuj polskie znaki w JS przed zapisem/szukaniem. WAL checkpoint przy SIGINT.
5. **Sync przez git**: commit tylko realnych zmian (diff-check, nie co tick); lock pid-file; skip gdy repo w merge/rebase. Sekrety NIGDY do paczki (scrubber + świadomość że łapie znane wzorce).
6. **Path traversal:** serwując pliki z dysku waliduj że ścieżka nie wychodzi poza katalog (`../`).
7. **Automat przetwarzający serię** (np. grafiki z pliku danych, import): kolejka w pliku JSON w repo ze statusem per rekord. Bierz 1 rekord na przebieg, `done` zapisuj DOPIERO po sukcesie, więc po awarii wznawiasz bez dubli. Harmonogram lokalny: `schtasks /SC DAILY /ST 08:00` na Windows, cron na Linuksie. Nie potrzebujesz lokalnych plików -> nie tu, patrz `stack-pitfalls` (automaty w tle).

## Anti-pattern → pattern
- ❌ `npm install express`. → ✅ wbudowany `http.createServer`.
- ❌ Dane w `localStorage`. → ✅ plik JSON/MD w repo (sync przeżyje).
- ❌ `@import` fontu z googleapis (apka offline pada bez netu). → ✅ woff2 lokalnie + `@font-face`.
- ❌ Serwowanie pliku z `url` bez walidacji → path traversal (`../../.ssh`). → ✅ resolve + sprawdź prefix katalogu.
- ❌ Autosync commituje co tick (pusty commit spam). → ✅ commit tylko gdy realny diff.

## Checklist końcowy
- [ ] Zero zależności runtime (działa bez npm install).
- [ ] Dane w plikach w repo; sekrety poza repo/paczką.
- [ ] SQLite: PL znaki znormalizowane; path traversal zablokowany.
- [ ] `start.bat` odpala jednym kliknięciem.
