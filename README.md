# claude-code-skills-pl

10 skilli do [Claude Code](https://code.claude.com/docs/en/skills) spisanych z produkcyjnej roboty: Next.js, Supabase, Stripe, Vercel i Cloudflare, mobile web, lokalne apki w Node oraz gry Roblox na Rojo. Każda reguła odpowiada na błąd albo pułapkę, na którą łatwo trafić w produkcji.

*English: ten Claude Code skills covering common production bugs and pitfalls (Next.js, Supabase RLS, Stripe payouts, Vercel/Cloudflare deploys, mobile web, Roblox/Rojo). Content is in Polish, trigger keywords in Polish and English.*

## Co jest w środku

| Skill | Kiedy się odpala |
|---|---|
| `stack-pitfalls` | Przed robotą w Supabase, PWA cache, mobile CSS, testach headless, deployu, Next.js 16, płatnościach, cronach. Lista objaw → fix, szczegóły w `references/`. |
| `supabase-rls-secdef` | Migracje, polityki RLS, funkcje SECURITY DEFINER, tabele z XP, kasą albo uprawnieniami. |
| `edge-function-auth` | Supabase Edge Functions, webhooki, wszystko na `service_role`. |
| `stripe-escrow-safety` | Webhooki Stripe, refundy, wypłaty, prowizje, ledger. Z sekcją o limitach Stripe w Polsce (BLIK, Przelewy24, 90 dni trzymania środków). |
| `web-frontend-hardening` | Render danych z bazy, inline `onclick`, Supabase realtime, Service Worker, publiczne formularze, zgody cookies. |
| `mobile-web-360` | Responsywne UI, PWA, iOS (zoom inputów, `dvh`, safe-area, web push). |
| `deploy-verify` | Po pushu z auto-deployem, zanim powiesz „działa na produkcji”. |
| `lokalna-apka-zero-deps` | Lokalna apka w Node bez zależności, dane w plikach, sync przez git. |
| `roblox-luau-rojo` | Gry Roblox pisane w plikach (Rojo, Wally, selene), testy bez Studio, logika po stronie serwera. |
| `writing-skills-for-weak-models` | Pisanie własnych skilli tak, żeby słabszy model (Haiku, Sonnet) wykonał je bez dopytywania. |

Każdy skill ma ten sam układ: kiedy używać i kiedy nie, twarde zasady, pary anti-pattern → pattern, przykład i checklistę na koniec.

## Instalacja

Skopiuj wybrane foldery z `skills/` do jednego z katalogów:

- `~/.claude/skills/` - skill działa we wszystkich projektach,
- `.claude/skills/` w repo projektu - tylko w tym projekcie.

```bash
git clone https://github.com/mauupa0/claude-code-skills-pl.git
cp -r claude-code-skills-pl/skills/stack-pitfalls ~/.claude/skills/
```

Claude Code wczyta skill w następnej sesji i odpali go sam, gdy zadanie pasuje do opisu we frontmatterze. Ręcznie: `/stack-pitfalls`.

## Zanim zaufasz

Część reguł zależy od dat i wersji (klucze API Supabase w 2026, wersje Next.js po łatkach CVE, wersje API Stripe). Stan wiedzy jest z jesieni 2026. Przed decyzją, która dotyczy pieniędzy albo bezpieczeństwa, sprawdź aktualną dokumentację.

## Licencja

MIT, szczegóły w [LICENSE](LICENSE).
