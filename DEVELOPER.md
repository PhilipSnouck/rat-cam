# Developer Documentation — rat-cam

Technische context voor een developer of LLM die aan deze codebase werkt.

---

## Wat dit is

Een volledig client-side PWA die één telefoon in een bewegingsgestuurde beveiligingscamera
verandert. Geen backend, geen database, geen API-keys, geen build step — alle logica zit in
één `index.html` (vanilla JS, ~220 regels script). Clips worden lokaal opgeslagen in IndexedDB.

- **Local:** `C:\Users\p.snouckaert\Own AI projects\rat-cam`
- **Deploy:** statische site op Vercel (framework preset **Other**, geen build). `vercel --prod` of via GitHub-import.
- **Taal:** UI en README zijn Nederlands.

---

## Stack

| Laag | Keuze | Reden |
|---|---|---|
| App | Eén `index.html` met vanilla JS + inline CSS | Geen build, geen dependencies, makkelijk te begrijpen |
| Camera | `getUserMedia` (video 1280×720 ideal + audio) | Vereist secure context (https of localhost) |
| Opname | `MediaRecorder`, chunks van 1s | Continue recorder maakt pre-roll mogelijk |
| Bewegingsdetectie | Canvas 96×72, grijswaarde-verschil per pixel per frame | Goedkoop genoeg voor `requestAnimationFrame`-loop |
| Opslag | IndexedDB (db `rat-cam`, store `clips`) + `navigator.storage.persist()` | Clips overleven herstart; persist beschermt tegen iOS 7-dagen-opruiming |
| Offline/PWA | `manifest.json` + `service-worker.js` (cache-first) | Installeerbaar, werkt offline na eerste load |
| Hosting | Vercel | https gratis (nodig voor camera), headers via `vercel.json` |

Geen environment variables, geen secrets. Er is niets te configureren buiten de app-UI.

---

## Bestandsstructuur

```
index.html          De hele app: UI, styling en logica in één bestand
manifest.json       PWA-manifest (standalone, portrait)
service-worker.js   Offline cache. Versietag CACHE = "rat-cam-v4"
vercel.json         Headers: service worker niet cachen + Permissions-Policy camera/mic
icon-192.png        App-icoon (ook apple-touch-icon)
icon-512.png        App-icoon groot
README.md           Gebruikersgerichte uitleg (NL)
```

---

## Hoe de onderdelen samenhangen (index.html)

1. **Detectieloop** — `loop()` draait permanent via `requestAnimationFrame`. Elke frame wordt
   het videobeeld op een 96×72 canvas getekend; per pixel wordt het RGB-somverschil met de
   vorige frame vergeleken (drempel 75). Het percentage veranderde pixels is het "bewegingsniveau".
2. **Drempel** — `triggerThreshold()` = `0.4 + (100 - gevoeligheid) * 0.14`. Gevoeligheid 100
   → drempel 0.4%, gevoeligheid 1 → ~14.3%. De meter schaalt op `METER_SCALE = 25` (25% niveau = volle balk).
3. **Pre-roll via continue recorder** — zodra de bewaking aan staat draait er altijd een
   `MediaRecorder` met `start(1000)`. Het **eerste** datablok wordt apart bewaard als `header`
   (init-segment); volgende blokken gaan in `buffer` met timestamp. De buffer wordt getrimd tot
   pre-roll + 4s marge.
4. **Capture** — bij beweging boven de drempel start `beginCapture()`: pre-roll-chunks worden
   uit de buffer gefilterd, nieuwe chunks gaan in `capChunks`. `scheduleStop()` herplant zich
   zolang er beweging blijft; harde max is `MAXCLIP = 180000` ms (3 min).
5. **Afronden** — `finishCapture()` plakt `header + prerollChunks + capChunks` tot één Blob en
   slaat die op via `saveClip()`. Daarna 2s cooldown (`cooldownUntil`) tegen her-triggeren.
6. **Opslag & lijst** — IndexedDB-records: `{id, ts, dur, size, blob, thumb}` (thumb is een
   96×120 JPEG-dataURL van het videobeeld). `renderClips()` bouwt de lijst; Bekijk speelt af in
   een `<dialog>`, Bewaar probeert eerst `navigator.share({files})` (deel-vel → cameraroll) en
   valt terug op een download-link, Wis verwijdert na `confirm()`.
7. **Wakkerhouden** — Screen Wake Lock zolang de bewaking aan staat; opnieuw aangevraagd bij
   `visibilitychange` naar visible. "Scherm op zwart" is alleen een zwarte overlay (`#blackout`)
   — camera en detectie lopen door.
8. **Veiligstellen** — bij `visibilitychange` naar hidden of `pagehide` wordt een lopende
   opname direct afgerond en opgeslagen.

---

## Lokaal draaien

Geen build. Serveer de map over http (localhost telt als secure context):

```bash
npx serve .          # of: python -m http.server
```

Open op `http://localhost:<poort>`. Let op: camera-permissie testen op een telefoon vereist
https — het makkelijkst via een Vercel-preview (`vercel`).

---

## Bekende eigenaardigheden / valkuilen

- **Cacheversie bumpen.** Bij elke wijziging aan de app: verhoog `CACHE` in `service-worker.js`
  (`rat-cam-v4` → `-v5`), anders blijven clients de oude versie uit de cache serveren. De
  service worker is cache-first; alleen het sw-bestand zelf wordt door `vercel.json`
  no-cache geserveerd.
- **Het header-chunk is heilig.** De pre-roll werkt alleen omdat het eerste MediaRecorder-blok
  (init-segment) vóór elke clip geplakt wordt. Sloop die logica niet; zonder header zijn de
  geplakte chunks niet afspeelbaar.
- **mp4 vs webm.** `pickMime()` prefereert webm (vp9 → vp8 → webm → mp4). Android levert webm,
  iOS Safari alleen mp4. Chunk-concatenatie is betrouwbaar op webm; op mp4 kan de getoonde
  duur/starttijd afwijken terwijl de beelden kloppen (staat ook in de README).
- **Achtergrond = einde opname.** Mobiele browsers pauzeren de camera bij achtergrond of een
  echt vergrendeld scherm; dat is niet te omzeilen. Vandaar: schermvergrendeling uit +
  "Scherm op zwart". De app rondt een lopende opname nog wel netjes af bij het wegnavigeren.
- **Testopname-truc.** `testBtn` forceert een capture door `lastMotion` ver in het verleden te
  zetten (`Date.now() - 1e9`), zodat `scheduleStop()` na precies de cliplengte stopt.
- **Detectie loopt altijd**, ook als de bewaking uit staat (de meter beweegt dan al) — alleen
  het triggeren van een opname is gekoppeld aan `armed`.
- **Geen audio-controle in de UI**: `getUserMedia` vraagt altijd ook de microfoon (clips bevatten
  geluid). `vercel.json` staat camera én mic toe voor self.
