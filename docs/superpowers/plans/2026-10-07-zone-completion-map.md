# ZoneCompletionMap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ESO-Addon, das auf Tamriel- und Aurbis-Karte vollständig abgeschlossene Zonen grün einfärbt.

**Architecture:** Reine Abschluss-Logik (`Completion.lua`, lokal getestet) + Umriss-Scanner über `GetMapMouseoverInfo` mit Session-Cache + Textur-Pool-Overlay über `ZO_WorldMapContainer`; Verdrahtung über Karten-Callbacks, Einstellungen via LibAddonMenu-2.0, Strings in 7 Sprachen.

**Tech Stack:** ESO Lua (5.1-Dialekt), LibAddonMenu-2.0, lokales `lua` (Homebrew) für Unit-Tests.

**Spec:** `docs/specs/2026-10-07-zone-completion-map-design.md`

## Global Constraints

- Addon-Dateien liegen ausschließlich unter `ZoneCompletionMap/` (Ordnername = Addon-Name, damit ESO es lädt); Tests und Docs außerhalb.
- Manifest: `## APIVersion: 101051 101052`, `## DependsOn: LibAddonMenu-2.0`, `## SavedVariables: ZoneCompletionMap_SV`, `## Author: Domenikus`, `## Version: 1.0.0`, `## AddOnVersion: 1`.
- Globaler Namespace nur `ZoneCompletionMap` (Tabelle); Module als Felder: `ZoneCompletionMap.Completion`, `.BlobScanner`, `.Overlay`, `.Settings`.
- Nur Lua-5.1-kompatible Syntax (kein `goto`, keine Integer-Division `//`, kein `utf8`).
- Eingefärbt nur bei `GetMapType()` == `MAPTYPE_WORLD` oder `MAPTYPE_COSMIC`.
- Standardfarbe `{ r = 0.2, g = 0.9, b = 0.3, a = 0.35 }`; Raster 50×50.
- Sprachen: `en de fr es ru jp zh`, Fallback `en`.
- Kein Lua-Fehler bei fehlenden Daten: `nil` aus API = 0, fehlender Umriss = ungefärbt.

## Review Focus

1. Zone ohne jede Aktivität in aktivierten Kategorien (z. B. alle Kategorien deaktiviert) → darf nicht grün werden. Test in Task 2.
2. Mehrere Treffer derselben Zone mit unterschiedlichen Texturen (Zone aus mehreren Blobs) → alle Teile färben, nicht nur den ersten. Test in Task 3 (Dedupe nur nach Textur, nicht nach zoneId).
3. `GetMapMouseoverInfo` liefert `mapId` = 0 / Zone ohne `zoneIndex` (Ozean, Hintergrund) → verwerfen, kein Fehler. Test in Task 3.
4. Wechsel Tamriel → Zonenkarte → zurück: Overlays dürfen auf der Zonenkarte nicht stehen bleiben. In-Game-Checkliste Task 6.
5. Zoomen während die Karte offen ist → Overlays bleiben deckungsgleich. In-Game-Checkliste Task 6.

---

### Task 1: Test-Harness und Projektgerüst

**Files:**
- Create: `ZoneCompletionMap/ZoneCompletionMap.txt`
- Create: `tests/run.lua` (Mini-Testrunner: `test(name, fn)`, `eq(a, b)`, Exit-Code 1 bei Fehlschlag)
- Create: `tests/smoke_spec.lua`

**Interfaces:**
- Produces: `tests/run.lua` — wird mit `lua tests/run.lua <spec-datei>...` aufgerufen, lädt jede Spec-Datei per `dofile`, Specs nutzen globale `test(name, fn)` und `eq(actual, expected, msg)`. Druckt `PASS name` / `FAIL name: msg`, am Ende `N passed, M failed`.

- [ ] **Step 1:** `brew install lua` (falls `lua -v` fehlt). Erwartet: `lua -v` zeigt eine Version.
- [ ] **Step 2:** `tests/smoke_spec.lua` mit `test("smoke", function() eq(1+1, 2) end)` schreiben.
- [ ] **Step 3:** `lua tests/run.lua tests/smoke_spec.lua` → schlägt fehl (run.lua fehlt).
- [ ] **Step 4:** `tests/run.lua` implementieren.
- [ ] **Step 5:** Erneut ausführen → `1 passed, 0 failed`, Exit 0.
- [ ] **Step 6:** Manifest mit den Feldern aus Global Constraints plus `## Title: Zone Completion Map` und Dateiliste in dieser Reihenfolge:
  ```
  lang/en.lua
  lang/$(language).lua
  Completion.lua
  BlobScanner.lua
  Overlay.lua
  Settings.lua
  ZoneCompletionMap.lua
  ```
- [ ] **Step 7:** Commit `chore: add test harness and addon manifest`.

### Task 2: Completion-Logik

**Files:**
- Create: `ZoneCompletionMap/Completion.lua`
- Test: `tests/completion_spec.lua`

**Interfaces:**
- Produces: `ZoneCompletionMap.Completion.IsZoneComplete(zoneId, isTypeEnabled, api) -> boolean`
  - `isTypeEnabled(completionType) -> boolean`
  - `api = { types = {completionType...}, getTotal = function(zoneId, type) -> number|nil, getCompleted = function(zoneId, type) -> number|nil }`
- Produces: `ZoneCompletionMap.Completion.DefaultApi() -> api` (im Spiel: `types = ZO_ZONE_STORY_ACTIVITY_COMPLETION_TYPES_SORTED_LIST`, Funktionen = `GetNum[Completed]ZoneActivitiesForZoneCompletionTypeAndIndex(zoneId, type, nil)`)
- Datei beginnt mit `ZoneCompletionMap = ZoneCompletionMap or {}`, damit sie standalone in Tests lädt.

- [ ] **Step 1: Tests schreiben** (Mock-API mit Typen `{1, 2, 3}`):
  - `"all complete"`: totals `{5,3,1}`, completed `{5,3,1}`, alle aktiv → `true`
  - `"one incomplete"`: completed `{5,2,1}` → `false`
  - `"incomplete type disabled"`: wie oben, Typ 2 deaktiviert → `true`
  - `"zero-total types ignored"`: totals `{5,0,1}`, completed `{5,0,1}` → `true`
  - `"nothing counts"`: alle Typen deaktiviert → `false`; und separat alle totals 0 → `false`
  - `"nil treated as zero"`: getTotal liefert `nil` für Typ 3, completed `nil` für Typ 3 → `true` (Typ 3 ignoriert)
  - `"nil completed with total"`: total 4, completed `nil` → `false`
- [ ] **Step 2:** `lua tests/run.lua tests/completion_spec.lua` → FAIL.
- [ ] **Step 3:** Implementieren.
- [ ] **Step 4:** Tests → alle PASS.
- [ ] **Step 5:** Commit `feat: add zone completion logic`.

### Task 3: BlobScanner

**Files:**
- Create: `ZoneCompletionMap/BlobScanner.lua`
- Test: `tests/blobscanner_spec.lua`

**Interfaces:**
- Produces: `ZoneCompletionMap.BlobScanner.Scan(api, gridSize) -> { {zoneId, texture, x, y, w, h}, ... }`
  - `api = { getNumLabels() -> n, getLabelPos(i) -> nx, ny, mouseover(nx, ny) -> name, texture, wN, hN, xN, yN, mapId, zoneIdForMapId(mapId) -> zoneId|nil }`
- Produces: `ZoneCompletionMap.BlobScanner.GetBlobs(mapId) -> list` — Cache pro `mapId` (Session-Tabelle), ruft beim ersten Mal `Scan(DefaultApi(), 50)`.
- Produces: `ZoneCompletionMap.BlobScanner.DefaultApi()` — `GetNumMapBlobs`, `GetMapBlobNameInfo(i)` (2./3. Rückgabe), `GetMapMouseoverInfo`, `zoneIdForMapId` = `GetZoneIndexByMapId` → `GetZoneId`, `nil` wenn zoneIndex nil/0 oder zoneId 0.
- Algorithmus: erst alle Label-Positionen, dann Raster-Zellmitten `(i - 0.5) / gridSize` für `i = 1..gridSize`; Treffer verwerfen bei leerer/`nil` Textur oder `zoneIdForMapId` = nil; Dedupe-Schlüssel = Texturpfad; Reihenfolge = erste Entdeckung.

- [ ] **Step 1: Tests schreiben** (Mock: Karte mit Rechtecken, `mouseover` gibt Textur des Rechtecks zurück, das den Punkt enthält):
  - `"finds blob via label"`: ein Blob, gridSize 1 verfehlt ihn, Label liegt drin → 1 Ergebnis mit korrekten `zoneId, texture, x, y, w, h`
  - `"finds blob via grid"`: keine Labels, gridSize 10 → gefunden
  - `"dedupes by texture"`: großer Blob, gridSize 10 → genau 1 Ergebnis
  - `"multiple textures same zone"`: zwei Blobs mit gleicher zoneId, unterschiedlichen Texturen → 2 Ergebnisse
  - `"skips invalid"`: Hintergrund liefert `""`-Textur bzw. mapId ohne Zone → 0 Ergebnisse, kein Fehler
- [ ] **Step 2:** Tests → FAIL.
- [ ] **Step 3:** `Scan` und `GetBlobs` implementieren (`DefaultApi`/`GetBlobs` nicht unit-getestet).
- [ ] **Step 4:** `lua tests/run.lua tests/*_spec.lua` → alle PASS.
- [ ] **Step 5:** Commit `feat: add map blob scanner`.

### Task 4: Lokalisierung

**Files:**
- Create: `ZoneCompletionMap/lang/en.lua`, `de.lua`, `fr.lua`, `es.lua`, `ru.lua`, `jp.lua`, `zh.lua`
- Test: `tests/lang_spec.lua`

**Interfaces:**
- Produces: String-IDs (via `ZO_CreateStringId` in `en.lua`, `SafeAddString(id, text, 1)` in den anderen):
  `ZCM_PANEL_TITLE` ("Zone Completion Map"), `ZCM_ENABLED` ("Enabled"), `ZCM_ENABLED_TOOLTIP` ("Tint fully completed zones on the Tamriel and Aurbis maps."), `ZCM_COLOR` ("Tint color"), `ZCM_COLOR_TOOLTIP` ("Color and opacity of the tint."), `ZCM_CATEGORIES` ("Categories"), `ZCM_CATEGORIES_DESC` ("A zone counts as complete when all enabled categories are done. Categories without activities in a zone are ignored."), `ZCM_DEBUG_HEADER` ("Zone Completion Map: <<1>> zones found"), `ZCM_DEBUG_LINE` ("<<1>> (<<2>>): <<3>>"), `ZCM_DEBUG_COMPLETE` ("complete"), `ZCM_DEBUG_INCOMPLETE` ("incomplete").
- Panel-Titel bleibt in allen Sprachen "Zone Completion Map".

- [ ] **Step 1: Test schreiben:** stubbt `ZO_CreateStringId(id, text)` / `SafeAddString(id, text, v)` als Tabellen-Rekorder, lädt `en.lua`, dann jede andere Sprachdatei einzeln; `"lang <code> covers all keys"` prüft, dass jede Sprache genau die Schlüsselmenge von `en` setzt und kein Text leer ist.
- [ ] **Step 2:** Test → FAIL.
- [ ] **Step 3:** Alle 7 Dateien schreiben (Übersetzungen natürlich formuliert; `<<1>>`-Platzhalter beibehalten).
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5:** Commit `feat: add localization for all ESO client languages`.

### Task 5: Overlay

**Files:**
- Create: `ZoneCompletionMap/Overlay.lua`

**Interfaces:**
- Produces: `ZoneCompletionMap.Overlay.Show(blobs, color)` (`blobs` = BlobScanner-Liste, `color = {r,g,b,a}`), `.Hide()`, `.Relayout()`.
- Umsetzung: lazily ein Root-Control `ZoneCompletionMapOverlay` (`CreateControl`, Parent `ZO_WorldMapContainer`, `SetAnchorFill`); Textur-Pool via `ZO_ControlPool` oder eigene Liste mit `WINDOW_MANAGER:CreateControl(nil, root, CT_TEXTURE)`; `SetDrawLayer(DL_BACKGROUND)` bzw. `SetDrawLevel(1)`, `SetBlendMode(TEX_BLEND_MODE_ALPHA)`; Pixelmaße aus `ZO_WorldMap_GetMapDimensions()`; Root-`OnUpdate` ruft `Relayout()` nur wenn sich die Dimensionen seit dem letzten Layout geändert haben.
- Nicht unit-getestet (reine UI); Verifikation in Task 6.

- [ ] **Step 1:** Implementieren.
- [ ] **Step 2:** `luac -p ZoneCompletionMap/Overlay.lua` → keine Ausgabe (Syntax ok).
- [ ] **Step 3:** Commit `feat: add world map overlay`.

### Task 6: Einstellungen, Verdrahtung, README

**Files:**
- Create: `ZoneCompletionMap/Settings.lua`, `ZoneCompletionMap/ZoneCompletionMap.lua`
- Modify: `README.md`

**Interfaces:**
- Consumes: `Completion.IsZoneComplete/DefaultApi`, `BlobScanner.GetBlobs`, `Overlay.Show/Hide`, String-IDs aus Task 4.
- Produces: `ZoneCompletionMap.Settings.Init(sv, onChange)` — LAM-Panel (`LibAddonMenu2:RegisterAddonPanel("ZoneCompletionMapPanel", {type="panel", name=GetString(ZCM_PANEL_TITLE), author="Domenikus", version="1.0.0", registerForDefaults=true})`), Controls: checkbox `enabled`, colorpicker `color` (mit Alpha), header `ZCM_CATEGORIES`, description `ZCM_CATEGORIES_DESC`, je checkbox pro Typ in `ZO_ZONE_STORY_ACTIVITY_COMPLETION_TYPES_SORTED_LIST` mit Name `GetString("SI_ZONECOMPLETIONTYPE", type)`, `getFunc` = `sv.types[type] ~= false`. Jeder `setFunc` ruft `onChange()`.
- `ZoneCompletionMap.lua`: `EVENT_ADD_ON_LOADED` → `ZO_SavedVars:NewAccountWide("ZoneCompletionMap_SV", 1, nil, defaults)`, `Settings.Init`, Callbacks `OnWorldMapChanged` + `WORLD_MAP_SCENE` `SCENE_SHOWING` → `Refresh()`, `SLASH_COMMANDS["/zcm"]` mit Argument `debug` (Ausgabe über `ZCM_DEBUG_*` + `zo_strformat`, Zonenname via `GetZoneNameById`). `Refresh()` exakt wie Spec-Abschnitt „ZoneCompletionMap.lua“.

- [ ] **Step 1:** Beide Dateien implementieren.
- [ ] **Step 2:** `luac -p ZoneCompletionMap/*.lua ZoneCompletionMap/lang/*.lua` → keine Ausgabe; `lua tests/run.lua tests/*_spec.lua` → alle PASS.
- [ ] **Step 3:** README: Kurzbeschreibung, Abhängigkeit LibAddonMenu-2.0, Installation (Ordner `ZoneCompletionMap` nach `Documents/Elder Scrolls Online/live/AddOns`), `/zcm debug`, Hinweis „Übersetzungen maschinell, Korrekturen willkommen“, In-Game-Testcheckliste aus der Spec plus Review-Focus-Punkte 4 und 5.
- [ ] **Step 4:** Commit `feat: add settings panel and wire up map refresh`.
- [ ] **Step 5: In-Game-Test (Nutzer):** Checkliste im README durchgehen; Befunde zurückmelden.
