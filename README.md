# rat-cam — bewegingsgestuurde camera (PWA)

Een zelfstandige Progressive Web App die één telefoon in een beveiligingscamera verandert.
Geen tweede toestel, geen account, geen abonnement. Alles draait lokaal in de browser.

## Wat het doet

- Camera draait continu zodra de bewaking aan staat.
- Bewegingsdetectie via beeldvergelijking op een klein canvas (96×72, grijswaarde-verschil per pixel).
- Bij beweging start automatisch een opname met **pre-roll**: de seconden vóór de beweging zitten erbij.
- Opname loopt door zolang er beweging is en stopt na de ingestelde cliplengte na de laatste beweging (max 3 min).
- Clips worden lokaal bewaard in **IndexedDB**; per clip is er Bekijk / Bewaar / Wis. "Bewaar" opent het deel-vel ("Bewaar video" → cameraroll); op desktop valt het terug op download.
- Opgeslagen clips overleven het sluiten van de app en een herstart. De app vraagt **persistente opslag** aan (`navigator.storage.persist()`) zodat de browser de clips niet wegruimt (o.a. de iOS-regel die data na ~7 dagen niet-gebruik wist). De opslagregel toont "beveiligd" als dat gelukt is. Een lopende opname wordt bij het sluiten/achtergronden nog veiliggesteld.
- "Scherm op zwart": zwarte overlay terwijl camera en detectie doorlopen (vereist dat schermvergrendeling uit staat).
- Installeerbaar als PWA via manifest + service worker; werkt offline na eerste load.

## Bestanden

- `index.html` — volledige app (UI + logica, geen build nodig)
- `manifest.json` — PWA-manifest
- `service-worker.js` — offline cache (versietag `rat-cam-v3`)
- `icon-192.png`, `icon-512.png` — app-iconen
- `vercel.json` — headers (service worker niet cachen, camera/microfoon toestaan)

## Deployen op Vercel

Statische site, geen build step. Framework preset: **Other**.

```bash
npm i -g vercel
vercel          # preview
vercel --prod   # productie
```

Of via GitHub: push deze map naar een repo en importeer 'm in Vercel.

## Belangrijk

- Camera-toegang vereist een **secure context** (https of localhost). Vercel levert https, dus dat is geregeld.
- Houd de app op de voorgrond met scherm aan. Mobiele browsers pauzeren de camera op de achtergrond of bij een echt vergrendeld scherm — dat kan een browser-app niet omzeilen. Daarom: automatische schermvergrendeling uit + "Scherm op zwart" gebruiken.
- iOS Safari neemt op als mp4 (Android: webm). Pre-roll wordt gemaakt door video-fragmenten samen te voegen; betrouwbaar op webm, meestal goed op mp4 (duur/starttijd kan soms afwijken terwijl de beelden kloppen).
- Bij het wijzigen van de app: verhoog de cacheversie in `service-worker.js` (`rat-cam-v1` → `-v2`) zodat clients de nieuwe versie ophalen.

## Instellingen in de app

- **Gevoeligheid** — hoger = reageert op kleinere beweging.
- **Cliplengte** — opnameduur na de laatste beweging (10–120 s).
- **Pre-roll** — seconden vóór de beweging die meegaan (0–10 s).
