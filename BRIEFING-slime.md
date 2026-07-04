# BRIEFING – Slime App (Übergabe an Claude Code)

> **Zweck dieses Dokuments:** Dich (Claude Code) ohne Chatverlauf arbeitsfähig machen.
> Lies das einmal komplett, bevor du Code anfasst. Nicole arbeitet über die
> GitHub-Weboberfläche und hat **keinen Terminal-Hintergrund** – wenn du ihr
> Schritte gibst, erkläre jeden Befehl und was danach passiert. Als Entwicklerin
> im Übrigen kompetent behandeln, nicht bevormunden. Sprache: **Deutsch, Du**.

---

## 1. Was ist das?

Eine eigenständige, spielerische **3D-Slime-App**: ein glänzend-durchsichtiger,
verschmelzender Gel-Klumpen mit Glitzer, den man per Maus/Touch ziehen und
drücken kann. Reines Privatprojekt (kein Finanzleser-Kontext), Ästhetik:
soft/girly/pastell. Läuft als **eine `index.html`** auf GitHub Pages.

**Status:** Erste vollständige Version fertig und funktionsfähig. Noch nicht deployed.

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
2. **Presets** – `PRESETS` Objekt mit `pink` / `holo` / `gold`
3. **Renderer / Szene / Kamera** – ACES-Tonemapping, PixelRatio auf max. 2 gedeckelt
4. **Environment** – RoomEnvironment via PMREM → `scene.environment`
5. **`makeBackground(inner, outer)`** – radialer Verlauf als CanvasTexture, wird
   pro Preset neu erzeugt und als `scene.background` gesetzt (das Gel bricht ihn)
6. **Licht** – ein `DirectionalLight` (Key) + ein farbiges `PointLight` (Rim)
7. **Slime** – `MeshPhysicalMaterial` + `MarchingCubes` (Objekt `marching`)
8. **Blob-Physik** – Array `blobs`, jeweils `{pos, vel, home, z, phase, strength}`
   im **0..1-Feldraum** (0.5 = Mitte)
9. **Glitzer** – BufferGeometry mit Attributen `aPhase/aSize/aHue` + eigener
   Vertex/Fragment-Shader (`sparkleMat`), additives Blending
10. **Pointer-Interaktion** – Pointer Events (Maus + Touch vereinheitlicht)
11. **`applyPreset(p)`** – setzt Material, Background, Rim-Licht, Glitzerfarbe
12. **Animationsloop** `render()` – Physik → `marching.reset()`/`addBall()`/`update()` → Render
13. **Resize** – Ortho-Frustum + Renderer neu setzen

---

## 5. Stellschrauben (KONFIG-Block, ganz oben)

| Konstante        | Default | Wirkung                                                        |
|------------------|---------|---------------------------------------------------------------|
| `RESOLUTION`     | 48      | Detailgrad der Oberfläche. **Bei iPhone-Ruckeln → 40.**       |
| `NUM_BLOBS`      | 6       | Anzahl verschmelzender Kugeln                                  |
| `NUM_SPARKLES`   | 260     | Glitzerpartikel                                               |
| `VIEW_SIZE`      | 4.4     | Sichtbare Höhe in Weltkoordinaten                            |
| `FIELD_SCALE`    | 1.55    | Physische Größe des Slime-Felds                              |

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
- **Blob-Positionen werden geklemmt** (`clamp` auf ~0.12..0.88). Ohne Clamp
  fliegen Blobs aus dem Feld und der Slime verschwindet stückweise.
- **`transmission` ist der teuerste Effekt.** Es rendert die Szene zusätzlich in
  einen Buffer. Das ist der Haupt-Performancekostenpunkt auf Mobilgeräten, nicht
  die Blob-Zahl. Reduktion → zuerst `RESOLUTION` und `transmission` senken.
- **Physik ist bewusst unterdämpft** (`vel.multiplyScalar(0.90)`) → das
  Nachwabbeln beim Loslassen. Höherer Faktor = träger/gummiger, niedriger =
  schneller ruhig.
- **`scene.background` wird pro Preset neu erzeugt.** Nicht per CSS am `<body>`,
  weil `transmission` einen echten Szenen-Hintergrund zum Brechen braucht.

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

- „Nur zuschauen"-Modus (ohne Interaktion, für Ambient/Deko)
- Zwei-Finger-Geste: Slime auseinanderziehen / teilen
- Sound (Web Audio API) beim Quetschen – Nicole nutzt das in ihrer Calma-App
- Fetterer/gröberer Glitzer als Preset-Option
- Grundfarbe frei wählbar statt drei feste Presets
- Leichte Sub-Surface-Tönung für „echteren" Gel-Look über `attenuationColor`/`thickness`
- Screenshot-/Teilen-Button

---

## 11. Nach jeder Änderung verifizieren

1. Lokal über HTTP-Server öffnen (§7), Konsole auf Fehler prüfen.
2. Ziehen/Loslassen testen – Slime muss zusammenhängen und zurückschnappen.
3. Alle drei Presets durchklicken – Farbe, Background und Glitzer müssen wechseln.
4. Fenster verkleinern/Handygröße prüfen (Resize + safe-area).
5. Wenn möglich auf dem iPhone gegentesten (Touch + Performance).
