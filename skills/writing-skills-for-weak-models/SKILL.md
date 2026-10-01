---
name: writing-skills-for-weak-models
description: "Jak pisać skille (SKILL.md) pod słabsze modele (Haiku/Sonnet) wg spec Anthropic. Use when creating or editing a SKILL.md, designing a skill for your skill library, or when the user mentions napisz skill, SKILL.md, skill dla modelu, frontmatter, biblioteka skilli. Meta-skill: zapewnia że nowe skille są deterministyczne, jednoznaczne i poprawnie ładowane."
---

# Pisanie skilli pod słabsze modele

Odbiorca skilla to model SŁABSZY niż autor. Cel: wykona zadanie bez zadawania pytań. Maksymalna jednoznaczność, checklisty zamiast prozy.

## Kiedy używać / kiedy NIE
- **UŻYJ:** tworzysz/edytujesz SKILL.md; projektujesz skill; utrzymujesz bibliotekę.
- **NIE używaj:** zwykłe kodowanie/pisanie nie-skilli.

## Spec Anthropic (zweryfikowane 2026-09-25, code.claude.com/docs/en/skills): twarde reguły
1. **Frontmatter: wymagane `name` + `description`.** `name` = lowercase-z-myślnikami, <64 znaki, MUSI = nazwa folderu. `description` <1024 znaki (limit Agent Skills; Claude Code tnie opis w liście na 1536), zawsze w podwójnym cudzysłowie: dwukropek ze spacją w wartości bez cudzysłowu to niepoprawny YAML, a przy końcach linii CRLF Claude Code gubi wtedy cały opis i pokazuje sam nagłówek. Plik z LF. Pola opcjonalne Claude Code (`disable-model-invocation`, `user-invocable`, `allowed-tools`, `argument-hint`, `paths`, `context`, `model`...) dodawaj tylko po sprawdzeniu w aktualnych docs; claude.ai i Skills API przyjmują tylko `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`.
2. **`name` musi sam mówić o triggerze.** Przy trybie `name-only` w `skillOverrides` model widzi WYŁĄCZNIE nazwę. `stripe-escrow-safety` mówi, kiedy go użyć; `helper-v2` nie mówi nic.
3. **`description` to najważniejsze pole przy trybie pełnym.** Musi zawierać CO robi **i KIEDY** (triggery). Wzór: „Use when the user mentions X, asks for Y, or a file of type Z is involved" + słowa-klucze jakich user/model realnie użyje (PL i EN). Kluczowy przypadek na początku (ucinanie). Opis kosztuje tokeny w KAŻDEJ sesji, więc bez waty.
4. **Progressive disclosure, 3 poziomy:** name+description (domyślnie w każdej sesji, ~100 tok; przy `name-only` tylko name, przy `user-invocable-only`/`off` nic) → body przy aktywacji (**<5000 tokenów**) → `references/*.md` tylko gdy potrzeba. Długie materiały → `references/`, NIE do body.
5. **Powtarzalne operacje → `scripts/`** (gotowy skrypt), nie opis „napisz skrypt który...".
6. **Tryb widoczności to część skilla.** Zdecyduj go razem z treścią (`skillOverrides` w `settings.json`): pełny opis, sama nazwa albo tylko ręczne wywołanie.
7. **Twarde reguły na górę body.** Po kompakcji CC dokleja ostatnie wywołanie skilla z zachowanym POCZĄTKIEM, max 5000 tok na skill i 25 000 łącznie (najstarsze wypadają), a lista opisów skilli nie wraca wcale. Zakazy i pułapki przed procedurą, nic krytycznego tylko w końcowej checkliście.
8. **Ukrycie skilla: `disable-model-invocation: true` (frontmatter) albo tryb `user-invocable-only`/`off` (`skillOverrides`), nigdy `user-invocable: false`.** To ostatnie tylko chowa skill z menu `/`: opis dalej kosztuje w każdej turze, model dalej go odpala. `disable-model-invocation` wyrzuca opis z kontekstu, ale blokuje też preload do subagentów i odpalenie z zadania cyklicznego: tylko dla skilli z efektem ubocznym albo wyłącznie ręcznych (deploy, wysyłka).
9. **`paths:` (globy) tylko dla skilla, którego trigger to ZAWSZE plik.** Z `paths` model ładuje skill sam wyłącznie przy pracy na pasujących plikach: rozmowa o planie bez otwartego pliku go nie odpali. Na skillach kasy i bezpieczeństwa (`stripe-escrow-safety`, `supabase-rls-secdef`, `edge-function-auth`) tylko z evalem wyzwalania przed i po: tam fałszywe odpalenie jest tańsze niż brak. Każdy inny skill: przed i po dodaniu `paths` test wyzwalania (prompt z pasującym plikiem i bez).
10. **`context: fork` tylko dla skilla z konkretnym zadaniem i wynikiem** (audyt, raport). Subagent nie widzi historii rozmowy, a skill z samymi wytycznymi zwraca w nim pusty wynik. `model:` bez `context: fork` zmienia model na resztę tury i zeruje cache promptu: nie dawaj go skillom odpalanym w grubej sesji.

## Struktura body (dla słabszego modelu)
- **Kiedy używać / kiedy NIE** (explicit, inaczej model odpala nie tam).
- **Procedura numerowana, deterministyczna** (krok po kroku, zero „zależnie od sytuacji").
- **Anti-pattern → pattern, min. 3 pary** (redukują halucynacje; najlepiej z realnych bugów).
- **1-2 few-shot** (wejście → oczekiwane wyjście).
- **Checklist końcowy** YES/NO („zanim oddasz, sprawdź: [ ]...").

## Anti-pattern → pattern
- ❌ `description: "Skill do bezpieczeństwa."` (zero triggerów). → ✅ „Use when reviewing auth/RLS/payments, or when the user mentions XSS, escrow, webhook..."
- ❌ Body 8000 tokenów z całą dokumentacją. → ✅ Body <5000; detale do `references/`.
- ❌ Proza „rozważ różne podejścia". → ✅ Numerowane kroki + checklist.
- ❌ „Napisz skrypt walidujący". → ✅ gotowy `scripts/validate.mjs`.
- ❌ `name: My_Skill` ≠ folder. → ✅ `name: my-skill` = folder `my-skill/`.
- ❌ Dopracowany opis i nazwa `utils`, a skill ląduje w `name-only`. → ✅ nazwa niesie trigger (`supabase-migrations-safe`).

## Few-shot
Wejście: „zrób skill o bezpiecznych migracjach Supabase".
Oczekiwane: folder `supabase-migrations-safe/SKILL.md`, `name: supabase-migrations-safe`, `description` z triggerami (migration, ALTER TABLE, DDL, RLS...), body: kiedy/nie + 5 kroków + 4 anti-pattern→pattern z realnych pułapek + 1 few-shot + checklist; długie przykłady SQL → `references/examples.md`; tryb: pełny opis, bo to skill bezpieczeństwa.

## Checklist końcowy (zanim zapiszesz skill)
- [ ] `name` = folder, lowercase-hyphen, <64, sam mówi o triggerze.
- [ ] `description` <1024, mówi KIEDY (triggery PL+EN), nie tylko CO.
- [ ] Pola frontmattera spoza `name`/`description` sprawdzone w aktualnych docs.
- [ ] Body <5000 tok; długie → references/, skrypty → scripts/.
- [ ] Kiedy/kiedy-NIE + procedura numerowana + ≥3 anti→pattern + few-shot + checklist.
- [ ] Tryb w `skillOverrides` wybrany z powodem.
- [ ] Przed pisaniem 3 scenariusze testowe (prompt → co ma się stać). Po napisaniu test w świeżej sesji ze skillem i bez: bez skilla wychodzi to samo → skill zbędny.
- [ ] W transkrypcie testu JEST wywołanie `Skill` (albo `mcp__...` przy MCP). PASS bez wywołania = porównałeś dwa identyczne agenty.
- [ ] Haiku wykona to bez pytań? Nie duplikuje innego skilla?
- [ ] Fakty techniczne zweryfikowane (web/kod), nie z pamięci.
