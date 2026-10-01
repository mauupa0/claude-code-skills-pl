---
name: stripe-escrow-safety
description: "Bezpieczna kasa - Stripe, escrow, webhooki, ledger, wypłaty. Use when writing or reviewing anything touching money: Stripe webhooks, escrow, refunds, payouts, commission/ledger, subscription trials, or when the user mentions płatność, wypłata, prowizja, escrow, chargeback, webhook, saldo. Błąd tu = czyjeś realne pieniądze. Zawiera anti-patterny typowych dziur w kodzie kasy (np. drenaż środków zwrotem)."
---

# Kasa: Stripe / escrow / ledger - bezpieczeństwo

Najwyższa stawka. Każda operacja na kasie: guard stanu + idempotencja + kwota liczona serwerowo z ledgera. Tryb pracy: testy na kilka sposobów + test adwersaryjny („jak bym to zdrenował?").

## Kiedy używać / kiedy NIE
- **UŻYJ:** webhooki Stripe, refund/charge/payout, prowizje, ledger, subskrypcje/trial, listy wypłat.
- **NIE używaj:** kod bez związku z pieniędzmi.

## Zasady żelazne
1. **Guard stanu na KAŻDEJ zmianie statusu zlecenia.** `update({status})` bez filtra na obecny status = retry/atak cofa stan. Zawsze `.in('status', [dozwolone_stany_wejściowe])`.
2. **Refund tylko z właściwego stanu.** Funkcja zwrotu musi wymagać statusu `confirmed_by_client`/`dispute_resolved` - inaczej klient curl'em wyciąga zwrot PRZED potwierdzeniem (drenaż escrow).
3. **Kwoty „tylko raz".** Pola typu `final_amount`/`amount` - po ustawieniu zablokuj zmianę (jak `amount_min/max`). Mutowalna kwota = manipulacja przed refundem.
4. **Webhook idempotentny + guard.** Stripe ponawia eventy do 3 dni. `payment_intent.succeeded` i potwierdzenie płatności → `.in('status',[...])`; dedup przez unikalny indeks (INSERT jako atomowy claim, nie select-then-act).
5. **Saldo aktualizuj RELATYWNIE, nie absolutnie.** `balance = balance + debt` (bez `.eq('balance', old)`), inaczej równoległy nowy dług skasuje spłatę lub optimistic-lock zablokuje spłatę → podwójne obciążenie karty.
6. **Wypłata NETTO z ledgera**, nie z sumy brutto. Claim przed akcją bankową: `update(paid_out).neq('payout_status','paid_out').select('id')` - 0 wierszy = „już wypłacone, NIE rób przelewu" PRZED instrukcją przelewu.
7. **Chargeback/dispute widoczny.** Obsłuż `charge.dispute.created` i `charge.refunded` → flaga blokująca wypłatę + mail do admina. Bez tego chargeback jest niewidoczny a wypłata idzie dalej.
8. **Bonusy w DB ↔ Stripe spójne.** Bonus do okresu próbnego zapisany tylko w DB nie ruszy Stripe (obciąży kartę wg swojego `trial_period_days`). Zaktualizuj subskrypcję w Stripe (`trial_end`).
9. **Abuse ekonomii:** nagroda za polecenie nie może przewyższać tego, co platforma realnie zarabia na poleconym (samofinansujący się abuse). Licz milestone'y od realnego przepływu (karta/escrow), nie gotówki z samym naliczonym długiem.
10. **Podwójny submit to sprawa serwera.** Zablokowany przycisk, szeregowa kolejka `useActionState` ani `abort` akcji (nie cofa zapisu, który już poszedł do bazy) NIE zastępują idempotencji. Chroni unikalny claim w RPC + `Idempotency-Key` w każdym POST do Stripe tworzącym albo zwracającym pieniądze (wzór: `refund-${order_id}` przy zwrocie). `useOptimistic` nigdy na saldo ani status płatności; kwota nigdy z hidden inputa, tylko z RPC/ledgera.

## Stripe w Polsce: limity i pułapki
1. **Stripe nie robi escrow.** Trzymanie kasy = separate charges and transfers (SC&T) + ręczne payouty, limit trzymania w PL **90 dni**. Dłużej = wymuszony payout albo refund, więc dodaj cron/alarm na płatność starszą niż ~75 dni bez rozliczenia. W UI i regulaminie pisz „płatność zabezpieczona do odbioru”, nie „escrow”.
2. **Hold (manual capture) tylko kartą, max 7 dni** (Visa/MC online; Visa MIT ok. 4 dni 18 h). BLIK i Przelewy24 na Stripe nie mają manual capture. Ustawiaj `payment_method_options[card][capture_method]=manual`, nigdy globalny `capture_method=manual` przy `automatic_payment_methods` (wytnie BLIK z listy metod). BLIK: tylko PLN, kod ważny 2 min, Express Checkout Element go nie obsługuje, recurring tylko w preview (abonament = karta).
3. **Transfer do sprzedawcy (SC&T, connected account):** `source_transaction` podaj przy tworzeniu transferu (później się nie da), inaczej transfer może paść na braku salda. Metody asynchroniczne (SEPA Debit): transfer dopiero po `charge.succeeded`, Stripe nie cofnie go sam, gdy płatność padnie. Refund po transferze idzie z salda PLATFORMY → zrób reversal transferu. Funds segregation nie obejmuje PL, saldo platformy jest wspólne dla wszystkich zleceń: przed transferem/refundem sprawdź saldo zlecenia w ledgerze.
4. **Przelewy24 przez Stripe: nie włączaj w marketplace, agencji IT ani szkole online** (zakazane MCC: marketplace, reklama/marketing, usługi IT, uczelnie). P24 nie ma też sporów, zwrot asynchroniczny do 3 dni roboczych. BLIK ma spory (12 dni na dowody): przy otwartym sporze nie wypłacaj sprzedawcy (zasada 7).
5. **Przypnij wersję API.** Raw `fetch` bez nagłówka `Stripe-Version` i webhook endpoint bez wersji używają wersji domyślnej konta: upgrade w Workbench zmienia naraz wszystkie edge functions i payloady webhooków. Dahlia `2026-08-26` usuwa `payment_method_types` z PaymentIntents/SetupIntents, a nowe platformy dostają błąd przy direct charges na legacy Express/Custom. Przed upgradem: nagłówek `Stripe-Version` w kodzie, grep po `payment_method_types`, rollback w Workbench do 72 h.
6. **Faktura B2B (np. prowizja platformy dla firmy) = KSeF**, Stripe jej nie wystawi. Nie pisz własnej integracji KSeF API (brak oficjalnego SDK JS/TS, XAdES, szyfrowanie, sesje): wołaj API programu do faktur z KSeF z edge function/route handlera. Środowisko testowe KSeF jest współdzielone: tylko fikcyjne NIP-y. Kogo dotyczy obowiązek: ksef.podatki.gov.pl.

## Anti-pattern → pattern
- ❌ Funkcja zwrotu bez guardu statusu, kwota mutowalna, trigger nadpisuje flagę zwrotu. → ✅ Refund tylko z `confirmed_by_client/dispute_resolved`; `final_amount` niezmienialna po ustawieniu; trigger nie zeruje flagi refundu.
- ❌ Pobieranie prowizji zeruje saldo absolutnie `.eq('balance', old.balance)` → nowy równoległy dług kasuje spłatę → podwójne obciążenie. → ✅ `balance = balance + debt` relatywnie.
- ❌ `payment_intent.succeeded` robi `update(status:'paid')` bez filtra → retry Stripe cofa zlecenie z późniejszego stanu. → ✅ `.in('status',['confirmed','paid'])`.
- ❌ `markPaidOut` `update('paid_out').eq('id')` bez guardu → dwie sesje = podwójny przelew. → ✅ `.neq('payout_status','paid_out').select('id')`, 0 wierszy = stop przed przelewem.
- ❌ Brak handlera chargeback → wypłata idzie mimo cofniętej płatności. → ✅ `charge.dispute.created` → blokada wypłaty + alert.

## Przykład (few-shot)
Wejście: „dodaj endpoint refundu dla zlecenia".
Oczekiwane: funkcja (1) autoryzuje właściciela (patrz edge-function-auth), (2) czyta zlecenie z DB, (3) `if (!['confirmed_by_client','dispute_resolved'].includes(status)) return 400`, (4) kwotę bierze z ledgera nie z body, (5) idempotencja (flaga refundu jako atomowy claim), (6) dopiero wtedy Stripe refund. Test adwersaryjny: curl w stanie `completed` → 400, nie zwrot.

## Checklist końcowy (kasa = 5×)
- [ ] Każda zmiana statusu ma `.in('status',[dozwolone])`.
- [ ] Refund tylko z potwierdzonego stanu; kwoty niezmienialne po ustawieniu.
- [ ] Webhook idempotentny (unique claim), guard stanu - retry = no-op.
- [ ] Saldo aktualizowane relatywnie; wypłata claim-przed-przelewem, netto z ledgera.
- [ ] Chargeback/dispute obsłużony i blokuje wypłatę.
- [ ] Bonusy spójne DB↔Stripe; bonus ≤ pobrana prowizja.
- [ ] Hold tylko per `payment_method_options[card]` (BLIK zostaje automatic); brak P24 w projektach z zakazanym MCC.
- [ ] `Stripe-Version` przypięty w kodzie, `Idempotency-Key` na każdym POST tworzącym albo zwracającym pieniądze, alarm na płatność > ~75 dni bez rozliczenia.
- [ ] Test adwersaryjny wykonany (curl z „złego" stanu → odrzucony).
