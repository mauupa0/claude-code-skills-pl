# Automaty w tle (cron, rutyny, alerty) - detale do stack-pitfalls

Czytaj, gdy stawiasz cron, rutynę Claude, monitoring z alertami albo integrację Google działającą bez Ciebie.

## 1. Gdzie odpalić automat
- Wynik liczony regułą (ping, diff, filtr, raport z plików) -> GitHub Actions cron + skrypt Node w repo. Zero tokenów, działa przy wyłączonym komputerze.
- Trzeba czytać albo pisać tekst -> rutyna `/schedule` (claude.ai/code/routines). Min. interwał 1 h, dzienny limit uruchomień na konto, zjada subskrypcję.
- Potrzebne lokalne pliki albo Studio -> Desktop scheduled task na komputerze, który stoi włączony.
- Tylko w otwartej sesji (np. pilnowanie deployu) -> `/loop`.
- Routing, wyceny, scoring = reguły w SQL albo Edge Function. LLM najwyżej do opisu.

## 2. Rutyna w chmurze: czego nie widzi
- Nie widzi lokalnych MCP (`claude mcp add`). Dodaj connector na claude.ai albo `.mcp.json` w repo.
- Środowisko Default = allowlista hostów. Inne hosty (Twoja domena, Supabase, własne API) dostają `403` z `x-deny-reason: host_not_allowed`. Fix: Network access: Custom + domeny.
- Zielony status = tylko brak błędu infrastruktury. Sukces zadania sprawdzasz w transkrypcie runu.
- Logika w skillu w repo, rutyna = sam wyzwalacz.

## 3. Rutyna działa bez pytania o zgodę
- Leci jako Ty (commity i wiadomości idą z Twoich kont), bez pytania o zgodę.
- Przy tworzeniu WSZYSTKIE connectory są zaznaczone domyślnie, łącznie z zapisem. Odznacz zbędne.
- Wszystko, co wychodzi na zewnątrz (mail, post, oferta), kończy się draftem w repo albo artefakcie. Wysyłka po ręcznym OK.
- GitHub odłączony > 72 h = rutyna się wyłącza. Po podpięciu włącz ją ręcznie.

## 4. GitHub Actions `schedule`
- Cron w UTC.
- O pełnej godzinie joby są kolejkowane, a przy obciążeniu DROPOWANE. Dawaj minuty typu `:05`, `:07`, `:17`.
- Min. interwał 5 min. Cron chodzi z ostatniego commita domyślnej gałęzi.
- Publiczne repo bez aktywności 60 dni = crony wyłączone.

## 5. Alert monitoringu tylko przy ZMIANIE stanu
- Wysyłaj tylko przy przejściu ok -> pad i pad -> ok. Poprzedni stan trzymaj w trwałym pliku albo tabeli (np. `status.json` jako `prev`).
- Timeout to osobny stan. Bez tego alert leci co przebieg albo jeden wolny ping robi falę alertów.
- Test: zły URL -> 1 alert, drugi przebieg -> 0 alertów.

## 6. Google OAuth w automacie
- Projekt OAuth w trybie Testing (External): refresh token wygasa po 7 dniach, automat na Gmailu, Drive albo YouTube pada po tygodniu. Fix: Publish App (Audience -> Publish). Ekran "unverified app" przy własnej apce to norma.
- Limit 100 refresh tokenów na konto Google na client ID. Nowy token bez ostrzeżenia unieważnia najstarszy.
- Dodanie zakresu (np. `gmail.send`) nie działa na starym tokenie: nowa zgoda z `prompt=consent`.
