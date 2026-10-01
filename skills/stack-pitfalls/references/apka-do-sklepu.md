# Apka do sklepu (Capacitor/Expo) - detale do stack-pitfalls

Czytaj, gdy web (Next/statyczny site) ma trafić do App Store albo Google Play.

## 1. Wybór drogi
- Kolejność: PWA -> Capacitor -> Expo. Expo tylko przy apce mobile-first, bo to przepisanie UI na React Native.
- Capacitor z `server.url` na zdalną domenę = odrzut Apple 4.2 ("repackaged website"). Bundluj statyki i dodaj min. jeden natywny plugin (push, aparat, share).
- Next w Capacitorze = `output: 'export'` (+ `images.unoptimized`): Server Actions, route handlers i ISR nie działają w bundlu. API zostaje na Vercel/Workers.
- Supabase OAuth w Capacitorze wymaga deep linku / custom scheme.
- iOS build wymaga Maca: chmurowy build (Codemagic, Ionic Appflow, EAS dla Expo) albo pożyczony Mac. Android buduje się z Windows albo w GitHub Actions.

## 2. Sklep a kasa i konto
- Usługi fizyczne (np. dostawa, naprawa, wizyta) idą Stripe'em poza IAP (Apple 3.1.3(e); Play też zwalnia usługi fizyczne).
- Subskrypcja cyfrowa w apce (np. konto premium) = obowiązkowo IAP / Play Billing (Apple 3.1.1).
- Nowe OSOBISTE konto Play: zamknięty test min. 12 testerów przez 14 dni ciągłych przed produkcją. Konto organizacji (D-U-N-S) zwolnione.
