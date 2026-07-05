# BRIEFING – Slime App (Übergabe an Claude Code)

> **Zweck dieses Dokuments:** Dich (Claude Code) ohne Chatverlauf arbeitsfähig machen.
> Lies das einmal komplett, bevor du Code anfasst. Nicole arbeitet über die
> GitHub-Weboberfläche und hat **keinen Terminal-Hintergrund** – wenn du ihr
> Schritte gibst, erkläre jeden Befehl und was danach passiert. Als Entwicklerin
> im Übrigen kompetent behandeln, nicht bevormunden. Sprache: **Deutsch, Du**.

---

## 1. Was ist das?

Eine eigenständige **3D-Slime-Meditationsapp**: ein großer, glänzend-durchsichtiger
Gel-Klumpen mit Glitzer **im Inneren**, den man per Maus/Touch drücken und kneten
kann – langsam, zäh, beruhigend. Kein hektisches Herumfliegen: der Slime bleibt
immer ein zusammenhängendes Stück. Reines Privatprojekt, Ästhetik:
soft/girly/pastell. Läuft als **eine `index.html`** auf GitHub Pages.

**Status:** Meditations-Überarbeitung fertig (ruhige Knet-Physik, Druck-Delle,
Atem-Modus, optionaler Klang, 4 Presets, weiche Farbübergänge). Alles liegt auf `main`.

---

## 2. Tech-Stack

- **Three.js r0.170**, geladen per CDN über eine `<script type="importmap">`
  (keine Build-Tools, kein npm, kein Bundler)
- **MarchingCubes** (`three/addons/objects/MarchingCubes.js`) – erzeugt die
  verschmelzende Slime-Oberfläche aus mehreren „Metaball"-Kugeln
- **RoomEnvironment** + **PMREMGenerator** – prozedurale Studio-Umgebung für
  Reflexe/Brechung (kein externes HDR nötig)
- **MeshPhysicalMaterial** mit `transmission`, `clearcoat`, `iridescence` – der
  realistische Gel-Look
- **OrthographicCamera** – bewusst gewählt, weil sie das Pointer→Welt-Mapping
  exakt linear macht (siehe §6)
- **Custom `ShaderMaterial` auf `THREE.Points`** – der funkelnde Glitzer
- Vanilla JS, keine Frameworks. UI-Chrome (Farb-Buttons) ist plain HTML/CSS.
- Font: **Fredoka** (Google Fonts) nur für die Mini-UI.

---

## 3. Dateistruktur

```
/index.html      <- alles inline (HTML + CSS + JS im <script type="module">)
```

Bewusst eine einzige Datei, damit GitHub Pages sie ohne Konfiguration ausliefert
und Nicole sie über die Weboberfläche hochladen kann.

---

## 4. Aufbau der index.html (Reihenfolge im `<script type="module">`)

1. **KONFIG-Block** – Konstanten ganz oben (siehe §5)
2. **Presets** – `PRESETS` Objekt mit `rosa` / `pink` / `lila` / `tuerkis` / `regenbogen`
   (girly & candy: bewusst hohe roughness, wenig clearcoat/envMap = NICHT metallisch)
3. **Renderer / Szene / Kamera** – ACES-Tonemapping, PixelRatio auf max. 2 gedeckelt
4. **Environment** – RoomEnvironment via PMREM → `scene.environment`
5. **Hintergrund** – EINE persistente CanvasTexture (`drawBackground()`), deren
   Farben beim Preset-Wechsel im Loop weich überblendet werden
6. **Licht** – ein `DirectionalLight` (Key) + ein farbiges `PointLight` (Rim)
7. **Slime** – `MeshPhysicalMaterial` + `MarchingCubes` (Objekt `marching`);
   Material-Werte werden beim Preset-Wechsel Richtung `matTar`/`colorTar` gelerpt
8. **Blob-Aufbau + Anker** – Array `blobs` (`{a0, offR, pos, vel, z, phase, strength}`)
   im **0..1-Feldraum** (0.5 = Mitte) + `anchor`/`anchorVel` (die ganze Slime-Masse)
9. **Glitzer** – BufferGeometry mit Attributen `aPhase/aSize/aHue` + eigener
   Vertex/Fragment-Shader (`sparkleMat`), additives Blending; Wolke liegt IM Slime
10. **Pointer-Interaktion** – Pointer Events (nur `isPrimary`), liefert `pointer`
    und `pressureRaw` (echte Force auf iPhone, sonst 0.5)
11. **Atem-Modus** – `updateBreath(t)`: 4s ein · 2s halten · 6s aus, Text in `#breath`
12. **Klang (optional)** – `ensureAudio()`: synthetisiertes Pad + Knet-Rauschen
    (Web Audio, keine Dateien), Master-Gain wird weich ein-/ausgeblendet
13. **`applyPreset(p)`** – setzt nur ZIELWERTE; das Überblenden passiert im Loop
14. **Physik** `updatePhysics()` – Anker-Modell, Druckaufbau, Dämpfung, Tempolimit
15. **Animationsloop** `render()` – Atem → Physik → MarchingCubes (+ Druck-Delle)
    → Preset-Lerp → Audio-Kopplung → Render
16. **Resize** – Ortho-Frustum, `computeAnchorRange()`, Renderer neu setzen

---

## 5. Stellschrauben (KONFIG-Block, ganz oben)

| Konstante         | Default | Wirkung                                                        |
|-------------------|---------|---------------------------------------------------------------|
| `RESOLUTION`      | 52      | Detailgrad der Oberfläche. **Bei iPhone-Ruckeln → 44.**       |
| `NUM_BLOBS`       | 7       | 1 Kern + 6 Ring-Kugeln                                        |
| `NUM_SPARKLES`    | 560     | Glitzerpartikel                                               |
| `VIEW_SIZE`       | 4.2     | Sichtbare Höhe in Weltkoordinaten                             |
| `FIELD_SCALE`     | 1.9     | Physische Größe des Slime-Felds                               |
| `PRESS_RADIUS`    | 0.30    | Radius, in dem der Fingerdruck Material verdrängt (Feldraum)  |
| `SPARKLE_RADIUS`  | 0.70    | Radius der Glitzerwolke (Welt) – klein genug, um IM Slime zu bleiben |
| `SLIME_WORLD_R`   | 0.95    | Annahme für den Slime-Radius → begrenzt via `computeAnchorRange()`, wie weit der Slime wandern darf |

`reduceMotion` respektiert `prefers-reduced-motion` (schaltet Ambient-Wabbeln
und Glitzer-Rotation ab).

---

## 6. Kritische Details – hier NICHT drüberstolpern

- **Feldraum ist 0..1, nicht Weltkoordinaten.** `marching.addBall(x,y,z,...)`
  erwartet Positionen in `[0,1]` mit `0.5` = Mitte. Die gesamte Blob-Physik läuft
  in diesem Raum. Wer in Weltkoordinaten rechnet, bricht den Slime.
- **Pointer→Feld-Mapping** hängt an der OrthographicCamera:
  `NDC → Welt (× VIEW_SIZE/aspect) → Feld (÷ 2·FIELD_SCALE + 0.5)`. Wenn du die
  Kamera auf Perspective umstellst, ist dieses Mapping falsch und muss über einen
  Raycaster/`unproject` neu gemacht werden. **Kamera-Wechsel = Mapping-Rewrite.**
- **Anker-Modell statt Einzelanziehung.** Die Blobs hängen an EINEM trägen
  Anker (`anchor`) mit festen, langsam rotierenden Offsets. Der Finger zieht
  nur den Anker, nie einzelne Blobs → der Slime kann prinzipbedingt nicht
  mehr auseinanderfliegen. Wer wieder Kräfte direkt auf einzelne Blobs gibt,
  holt das „Explodieren" zurück.
- **Druck = negative Metaball-Kugel.** `marching.addBall(..., -1.0*press, ...)`
  am Finger drückt die Oberfläche sichtbar ein (MarchingCubes unterstützt
  negative Stärken über `Math.sign`). Dazu werden Blobs im `PRESS_RADIUS`
  sanft zur Seite gedrückt – das ist das Knet-Gefühl. `press` baut sich weich
  auf/ab; auf iPhones fließt die echte Touch-Force (`e.pressure`) ein.
- **Tempolimit + Dämpfung.** `vel` wird pro Frame ×0.85 gedämpft UND auf
  max. 0.015 Länge geklemmt. Das Limit ist die zweite Explosions-Sicherung.
  Der Slime ist bewusst ÜBERdämpft (zäh, kriecht langsam zurück, wackelt
  nicht nach); `press` baut sich schnell auf, aber langsam ab (Delle erholt
  sich träge). Der Anker bleibt an Ort und Stelle und "lehnt" sich nur
  max. `ANCHOR_LEAN` (0.03) zum Finger – der Slime wandert nicht mehr.
- **Blob-Positionen werden geklemmt** (`clamp` auf ~0.14..0.86). Ohne Clamp
  fliegen Blobs aus dem Feld und der Slime verschwindet stückweise. Der Anker
  wird zusätzlich über `computeAnchorRange()` (aspect-abhängig!) begrenzt,
  damit der Slime auf schmalen Handys nie den Bildschirm verlässt.
- **`transmission` ist der teuerste Effekt.** Es rendert die Szene zusätzlich in
  einen Buffer. Das ist der Haupt-Performancekostenpunkt auf Mobilgeräten, nicht
  die Blob-Zahl. Reduktion → zuerst `RESOLUTION` und `transmission` senken.
- **Preset-Wechsel setzt nur Ziele.** `applyPreset()` schreibt in `matTar`,
  `colorTar`, `bg*Tar` usw.; der Loop lerpt jeden Frame dorthin. Direkt am
  Material schreiben bricht das weiche Überblenden.
- **`scene.background` ist EINE persistente CanvasTexture,** die bei Farbwechsel
  neu gezeichnet wird (`drawBackground()`). Nicht per CSS am `<body>`, weil
  `transmission` einen echten Szenen-Hintergrund zum Brechen braucht.
- **Audio erst nach User-Geste.** `ensureAudio()` läuft im Klick-Handler des
  Klang-Buttons (Autoplay-Policy). Alles synthetisiert, keine Dateien.

---

## 7. Lokal testen (WICHTIG)

ES-Module + importmap funktionieren **nicht** über `file://` – ein direkter
Doppelklick auf die HTML-Datei zeigt eine leere/schwarze Seite. Es braucht einen
lokalen HTTP-Server. Auf Nicoles Windows-Rechner z.B.:

```
python -m http.server 8000
```

…im Ordner der `index.html`, dann im Browser `http://localhost:8000`.
(Falls du ihr das gibst: erklären, dass das Fenster offen bleiben muss und man
es mit Strg+C stoppt.) Alternativ VS Code „Live Server"-Extension.

---

## 8. Deployment (GitHub Pages, Nicoles Weg)

GitHub: `nicolehahn2890`. Ablauf über die **Weboberfläche** (kein git-CLI):

1. Neues Repo anlegen, Vorschlag: `Slime` (Repo-Name → URL-Slug anpassen)
2. `index.html` über „Add file → Upload files" hochladen, committen
3. **Settings → Pages** → Source: Branch `main`, Ordner `/ (root)` → Save
4. Nach ~1 Min live unter `https://nicolehahn2890.github.io/Slime/`

---

## 9. Konventionen (Nicoles Präferenzen)

- Code **vollständig und sofort lauffähig**, keine Platzhalter, keine „…"-Auslassungen.
- Kommentare auf Deutsch, knapp.
- Design privat: soft/girly/pastell. Maskottchen ihrer anderen Apps ist „Rex der
  T-Rex" – hier bisher nicht verwendet, nur einbauen wenn explizit gewünscht.
  **Keine Dino-Emojis** ohne ausdrückliche Bitte.
- Bei größeren Umbauten erst Annahmen offenlegen, dann bauen.

---

## 10. Backlog / mögliche nächste Schritte (nicht beauftragt, nur Ideen)

- ~~Sound (Web Audio API) beim Quetschen~~ ✅ eingebaut (Pad + Knet-Rauschen, Toggle im Dock)
- ~~„Nur zuschauen"/Ambient~~ ✅ als Atem-Modus eingebaut (4·2·6-Rhythmus)
- Zwei-Finger-Geste: Slime auseinanderziehen / teilen
- Fetterer/gröberer Glitzer als Preset-Option
- Grundfarbe frei wählbar statt fünf feste Presets
- Vibration (Haptics API) beim Kneten auf Android
- Screenshot-/Teilen-Button
- Atem-Rhythmus wählbar (Box-Breathing 4·4·4·4, 4-7-8 …)

---

## 11. Nach jeder Änderung verifizieren

1. Lokal über HTTP-Server öffnen (§7), Konsole auf Fehler prüfen.
2. Ziehen/Loslassen testen – Slime muss zusammenhängen und zurückschnappen.
3. Alle drei Presets durchklicken – Farbe, Background und Glitzer müssen wechseln.
4. Fenster verkleinern/Handygröße prüfen (Resize + safe-area).
5. Wenn möglich auf dem iPhone gegentesten (Touch + Performance).
