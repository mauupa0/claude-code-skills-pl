---
name: supabase-rls-secdef
description: "Bezpieczne wzorce Supabase RLS + SECURITY DEFINER RPC (progresja, XP, kasa, ledger, uprawnienia). Use when writing or reviewing Supabase migrations, RLS policies, SECURITY DEFINER functions, or any table storing progression/XP/money/permissions; when the client (browser) could insert/update sensitive rows; or when the user mentions RLS, policy, GRANT/REVOKE, anti-cheat, ledger, „klient nie może pisać”. Zawiera anti-patterny typowych dziur."
---

# Supabase RLS + SECURITY DEFINER - sprawdzone wzorce

Cichy błąd w RLS/SECDEF = wyciek cudzych danych, podrobione zaliczenie albo drenaż kasy. Klient (przeglądarka) jest NIEZAUFANY - każdy request można podrobić curl'em z tokenem usera.

## Kiedy używać / kiedy NIE
- **UŻYJ:** migracje z tabelami progresji/XP/kasy/uprawnień; polityki RLS; funkcje SECDEF; gdy klient mógłby pisać do wrażliwej tabeli.
- **NIE używaj:** czysto publiczne read-only dane bez wrażliwości; logika czysto frontendowa.

## Zasady żelazne
1. **Klient NIGDY nie pisze wprost do tabel progresji/kasy/uprawnień.** Tylko przez `SECURITY DEFINER` RPC z walidacją SERWEROWĄ. Klient liczący wynik i wołający `submit_quiz(pct)` = klient decyduje o zaliczeniu = oszustwo.
2. **SECDEF waliduje DOWÓD, nie deklarację.** Funkcja musi sama sprawdzić że warunek zaszedł (np. sama ocenić test / sprawdzić że lekcja przerobiona), nie ufać parametrom od klienta.
3. **W SECDEF tożsamość bierz z `auth.uid()` / GUC, NIGDY z `current_user`** - w SECDEF `current_user` to zawsze właściciel funkcji (owner), więc check bypassu przez `current_user` zawsze przechodzi.
4. **RLS = osobna polityka per komenda** (SELECT / INSERT / UPDATE / DELETE). Polityka tylko na SELECT, a UPDATE otwarty = dziura.
5. **Eskalacja uprawnień:** polityka self-update na tabeli `profiles`/`permissions` MUSI wykluczać kolumnę roli - inaczej user ustawi sobie `role='admin'`. `WITH CHECK` blokuj zmianę roli.
6. **REVOKE kolumny łamie polityki:** `REVOKE SELECT(kolumna)` → każda polityka/trigger czytający tę kolumnę pada 42501. Po table-GRANT rób column-GRANT dla każdej nowej czytanej kolumny.
7. **Rekursja polityk (42P17):** polityka A czyta tabelę B której polityka czyta A → wynieś logikę do SECDEF helpera, polityka woła helper.
8. **Anti-replay:** RPC zmieniające saldo/XP → `pg_advisory_xact_lock(user_id)` + idempotencja, żeby równoległe wywołania nie zdublowały.
9. **GRANT jawnie w każdej migracji z CREATE TABLE.** Od 30.10.2026 nowa tabela w `public` NIE dostaje praw w Data API dla `anon`, `authenticated` ani `service_role` (edge functions na service_role też jej nie widzą); istniejące tabele zachowują granty. Szablon: `revoke all on public.t from anon, authenticated` → `grant` tylko potrzebnych czasowników per rola (service_role też) → `enable row level security` → polityki `TO authenticated`. Tabele kasy: authenticated tylko `select` albo nic. "permission denied" na nowej tabeli = najpierw granty, potem polityki. Branch bez historii migracji z CLI nie ma default privileges.
10. **Każda SECDEF: ustawiony `search_path` + odebrany EXECUTE od PUBLIC.** Nowe funkcje: `set search_path = ''` + nazwy w pełni kwalifikowane (`public.t`, `auth.uid()`); w projekcie, który ma już regułę `= public`, trzymaj się jej, nie przepisuj masowo. Postgres daje `EXECUTE TO PUBLIC`, więc revoke od samego anon niczego nie zamyka: `revoke execute on function f(args) from public, anon, authenticated`, potem `grant execute` tylko właściwej roli. Wyjątek: helper używany w politykach RLS (`is_admin`) zostaje wykonywalny dla roli, która ewaluuje politykę. Dowód: POST na `/rest/v1/rpc/<f>` z samym kluczem anon w `apikey` (bez JWT usera) zwraca `42501` permission denied; request bez `apikey` dostaje 401 od bramki zawsze = fałszywy PASS.

## Anti-pattern → pattern
- ❌ `submit_quiz(p_quiz_id, p_pct)` sprawdza tylko auth+istnienie quizu; klient sam liczy pct i pass. → ✅ Serwer (SECDEF/endpoint) sam ocenia odpowiedzi, sam liczy pct, sam stempluje zaliczenie; `submit_quiz` dla `authenticated` wycofany.
- ❌ `finish_lesson(p_lesson_id)` sprawdza tylko czy lekcja ISTNIEJE i wpisuje `completed` → podrobione zaliczenie bez robienia lekcji. → ✅ Wydanie zaliczenia związane z serwerowym dowodem (token/HMAC z ocenionego testu), nie z samym wywołaniem.
- ❌ Bypass w SECDEF sprawdzany `IF current_user = 'owner'`. → ✅ `IF auth.uid() = row.user_id` / GUC `request.jwt.claims`.
- ❌ Polityka `FOR SELECT USING (true)` na `profiles`, brak polityki UPDATE → domyślnie zablokowane... albo za szeroka self-update pozwala zmienić `role`. → ✅ `FOR UPDATE USING (id=auth.uid()) WITH CHECK (id=auth.uid() AND role = (SELECT role FROM profiles WHERE id=auth.uid()))` - rola niezmienialna.
- ❌ Test RLS przez samo czytanie polityk albo `SET LOCAL request.jwt.claims` bez zmiany roli i bez transakcji. → ✅ `begin; set local role authenticated; set local request.jwt.claim.sub = '<uuid>'; <realny SELECT/UPDATE>; rollback;`. Bez `begin` i `set local role` zapytanie leci jako postgres (omija RLS) = fałszywy PASS. Przed asercją "nie widzi" zaseeduj rekord INNEGO usera. pgTAP: `supabase/tests` + `supabase test db --db-url <DEV>`, nigdy `--linked` na prod.

## Przykład (few-shot)
Wejście: „dodaj RPC do przyznania 50 XP za ukończenie zadania".
Oczekiwane: SECDEF funkcja która (1) `advisory_xact_lock(auth.uid())`, (2) sama sprawdza w DB że zadanie zaliczone przez tego usera i XP jeszcze nienaliczone (idempotencja), (3) dopisuje do ledgera XP, (4) `GRANT EXECUTE ... TO authenticated`; klient NIE dostaje `UPDATE` na tabeli XP. Do tego test role-sim z rollbackiem.

## Checklist końcowy
- [ ] Żadna wrażliwa tabela nie ma klientowi otwartego INSERT/UPDATE (tylko przez SECDEF RPC).
- [ ] SECDEF używa `auth.uid()`/GUC, nie `current_user`; waliduje dowód nie deklarację.
- [ ] Polityki: osobna per komenda; self-update nie pozwala zmienić roli/salda.
- [ ] Zmiana kolumn: column-GRANT po REVOKE; brak rekursji polityk.
- [ ] RPC na saldo/XP: advisory lock + idempotencja.
- [ ] Przetestowane role-sim (`begin` + `set local role authenticated` + `jwt.claim.sub` + rollback, zaseedowany rekord innego usera), nie tylko odczyt polityk.
- [ ] Nowa tabela: jawne REVOKE/GRANT per rola (też service_role); nowa SECDEF: `search_path` ustawiony, EXECUTE odebrany od PUBLIC.
- [ ] Po migracji `supabase db advisors` (security): zero 0013 (RLS off), 0010 (widok omija RLS, fix `security_invoker=on`), 0011 (search_path), 0015 (RLS po `user_metadata`), 0028/0029 (SECDEF wołalna przez anon/authenticated).
