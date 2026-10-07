# ZoneCompletionMap – Design

Datum: 2026-10-07
Ziel-API: 101051 / 101052

## Ziel

Ein ESO-Addon, das auf der Weltkarte alle Zonen, die der aktuelle Charakter zu
100 % abgeschlossen hat, dezent grün einfärbt. So ist auf einen Blick sichtbar,
wo noch etwas zu tun ist.

## Anforderungen

- Eingefärbt wird auf den Übersichtskarten:
  - **Tamriel** (`MAPTYPE_WORLD`)
  - **Aurbis / kosmische Karte** (`MAPTYPE_COSMIC`) mit Kalthafen, Stadt der
    Uhrwerke, Fargrave, Apokryphen usw.
- Zonen- und Unterzonenkarten werden nicht eingefärbt.
- "Abgeschlossen" bedeutet: Alle **aktivierten** Kategorien des Zonenführers sind
  vollständig erledigt. Dazu gehört die Zonen-Story (Prioritätsquests).
- Jede Kategorie ist im Einstellungsmenü einzeln aktivierbar.
- Farbe, Deckkraft und ein globaler Ein/Aus-Schalter sind einstellbar.
- Das Einstellungsmenü nutzt LibAddonMenu-2.0 (Pflichtabhängigkeit).
- Alle eigenen Texte gibt es in allen ESO-Clientsprachen: `en`, `de`, `fr`, `es`,
  `ru`, `jp`, `zh`. Fallback ist `en`.
- Unterstützt wird nur die Tastatur/Maus-UI, Gamepad ist nicht Teil von v1.

## Verwendete API

Alles verifiziert gegen `esoui/esoui` (Branch `live`, API 101051).

| Zweck | Funktion |
|---|---|
| Aktivitäten gesamt | `GetNumZoneActivitiesForZoneCompletionTypeAndIndex(zoneId, type, index\|nil)` |
| Aktivitäten erledigt | `GetNumCompletedZoneActivitiesForZoneCompletionTypeAndIndex(zoneId, type, index\|nil)` |
| Kategorien (sortiert) | `ZO_ZONE_STORY_ACTIVITY_COMPLETION_TYPES_SORTED_LIST` |
| Kategorie-Name | `GetString("SI_ZONECOMPLETIONTYPE", type)` (vom Spiel lokalisiert) |
| Kartentyp | `GetMapType()` |
| Umriss an Punkt | `GetMapMouseoverInfo(nx, ny)` → `name, texture, wN, hN, xN, yN, mapId` |
| Beschriftungen | `GetNumMapBlobs()`, `GetMapBlobNameInfo(i)` → `name, nx, nz, nWidth, scale` |
| mapId → zoneId | `GetZoneIndexByMapId(mapId)` → `GetZoneId(zoneIndex)` |
| Kartengröße | `ZO_WorldMap_GetMapDimensions()` |
| Sprache | `GetCVar("language.2")` |

Die Zonen-Story entspricht `ZONE_COMPLETION_TYPE_PRIORITY_QUESTS`. Sie ist damit
eine normale Kategorie wie alle anderen.

## Architektur

```
ZoneCompletionMap/
  ZoneCompletionMap.txt   Manifest (DependsOn: LibAddonMenu-2.0, SavedVariables)
  lang/en.lua             Strings (Fallback, wird immer zuerst geladen)
  lang/$(language).lua    Strings der Clientsprache (überschreibt en)
  Completion.lua          Abschluss-Logik, rein, API wird injiziert
  BlobScanner.lua         Findet Zonen-Umrisse einer Karte, Cache pro mapId
  Overlay.lua             Textur-Pool über ZO_WorldMapContainer
  Settings.lua            LibAddonMenu-Panel
  ZoneCompletionMap.lua   Init, Events, Verdrahtung, Slash-Befehl
tests/
  completion_spec.lua     Unit-Tests für Completion.lua (Standard-Lua + Mocks)
```

### Completion.lua

`Completion.IsZoneComplete(zoneId, enabledTypes, api) -> boolean`

- Iteriert über `ZO_ZONE_STORY_ACTIVITY_COMPLETION_TYPES_SORTED_LIST`.
- Übersprungen werden deaktivierte Typen und Typen mit `total == 0`.
- Für jeden übrigen Typ gilt: `completed >= total`, sonst ist die Zone nicht
  abgeschlossen.
- Zählt in der Zone keine einzige aktivierte Kategorie, gibt die Funktion `false`
  zurück. Solche Zonen werden nie eingefärbt.
- Die Zählung erfolgt mit `index = nil`, also als Gesamtwert über alle Indizes,
  so wie die Fortschrittsanzeige im Spiel.
- `api` ist eine Tabelle mit den beiden Zählfunktionen. Im Spiel sind das die
  echten Funktionen, in den Tests Mocks.

### BlobScanner.lua

`BlobScanner.GetBlobs(mapId) -> { {zoneId, texture, x, y, w, h}, ... }`
(alle Werte normalisiert 0..1)

1. **Cache:** Liegt für `mapId` schon ein Ergebnis vor, wird es zurückgegeben.
   Der Cache lebt nur in der Session und wird nicht gespeichert.
2. **Beschriftungs-Sonden:** Für jeden Eintrag aus `GetMapBlobNameInfo(i)` wird
   `GetMapMouseoverInfo` an dessen Position abgefragt.
3. **Raster:** 50×50 Punkte mit Zellmitte bei `(i+0.5)/50`, jeweils
   `GetMapMouseoverInfo`.
4. Treffer ohne Textur oder ohne gültige `zoneId` werden verworfen.
5. Doppelte Treffer werden über den Texturpfad zusammengefasst, der erste
   Treffer gilt.

### Overlay.lua

- Pool aus `CT_TEXTURE`-Kontrollen, Kinder von `ZO_WorldMapContainer`.
- `DrawLevel` liegt unter dem nativen Hover-Umriss (Level 2). Der native
  Hover-Effekt bleibt also sichtbar.
- `Overlay.Show(blobs, color)`:
  - Jede Textur bekommt `SetTexture`, `SetColor(r, g, b, a)` und
    `SetDimensions(w*MAP_W, h*MAP_H)`.
  - Verankert wird mit `TOPLEFT` am Container, Offset `(x*MAP_W, y*MAP_H)`.
- `Overlay.Hide()` versteckt alle Texturen.
- `Overlay.Relayout()` rechnet Größe und Position neu, wenn sich
  `ZO_WorldMap_GetMapDimensions()` ändert.
  - Geprüft wird in einem `OnUpdate` des Overlay-Containers, nur solange die
    Karte sichtbar ist und sich die Größe geändert hat.
  - Verschieben der Karte braucht nichts, weil die Texturen Kinder des
    Containers sind.

### ZoneCompletionMap.lua

- `EVENT_ADD_ON_LOADED`: SavedVariables laden und Einstellungsmenü registrieren.
- Neu gezeichnet wird bei:
  - `CALLBACK_MANAGER:RegisterCallback("OnWorldMapChanged", Refresh)`
  - `WORLD_MAP_SCENE` StateChange → `SCENE_SHOWING`: `Refresh`
  - jeder Änderung einer Einstellung
- `Refresh()`:
  - Ist das Addon deaktiviert oder `GetMapType()` weder WORLD noch COSMIC,
    wird nur `Overlay.Hide()` aufgerufen.
  - Sonst werden die Blobs geholt, nach `IsZoneComplete` gefiltert und mit
    `Overlay.Show` gezeichnet.
- `/zcm debug` gibt pro gefundener Zone Name, `zoneId` und den Status
  abgeschlossen ja/nein im Chat aus.

### Settings.lua

SavedVariables sind account-weit: `ZO_SavedVars:NewAccountWide("ZoneCompletionMap_SV", 1, nil, defaults)`

Standardwerte:

```lua
{
  enabled = true,
  color = { r = 0.2, g = 0.9, b = 0.3, a = 0.35 },
  types = {}, -- [completionType] = false bedeutet deaktiviert; fehlend = aktiv
}
```

Panel-Inhalt:

- Checkbox "Aktiviert"
- Farbwähler mit Alpha
- Überschrift "Kategorien"
- je eine Checkbox pro Typ, Label `GetString("SI_ZONECOMPLETIONTYPE", type)`

## Lokalisierung

- Eigene Strings, z. B. Panel-Titel, "Aktiviert", "Farbe", Überschrift
  "Kategorien", Tooltips und Debug-Ausgaben, werden über `ZO_CreateStringId` /
  `SafeAddString` angelegt.
- `lang/en.lua` wird immer geladen. Danach lädt `lang/$(language).lua` die
  Clientsprache und überschreibt die englischen Texte mit höherer Version.
- Es gibt eine Datei pro Sprache: `en`, `de`, `fr`, `es`, `ru`, `jp`, `zh`.
- Kategorienamen kommen aus dem Spiel und sind dadurch automatisch korrekt
  lokalisiert.
- Die Übersetzungen erstelle ich (maschinell). Sie sollten von Muttersprachlern
  geprüft werden, das wird im README vermerkt.

## Fehlerverhalten

- Ein nicht gefundener Umriss bedeutet: Die Zone bleibt ungefärbt. Es gibt keinen
  Lua-Fehler.
- Liefert eine API-Funktion `nil`, wird das wie `0` behandelt.
- Fehlt LibAddonMenu, lädt ESO das Addon nicht und zeigt die fehlende
  Abhängigkeit an (`DependsOn`).

## Tests

- `tests/completion_spec.lua` läuft mit Standard-Lua (`lua tests/completion_spec.lua`)
  und eigenen Mini-Asserts, ohne externe Abhängigkeit. Abgedeckte Fälle:
  - alle Typen komplett → true
  - ein Typ unvollständig → false
  - der unvollständige Typ ist deaktiviert → true
  - keine aktivierten Typen mit Aktivitäten → false
  - API liefert `nil` → wird wie 0 behandelt
- Manuelle In-Game-Checkliste in `README.md`:
  - Tamriel-Karte: abgeschlossene Zone ist grün, offene Zone ist nicht grün
  - Aurbis-Karte: Kalthafen und Stadt der Uhrwerke werden korrekt behandelt
  - Zoomen und Verschieben: Overlays bleiben deckungsgleich
  - Hover über eine grüne Zone: der native Hover-Effekt ist weiter sichtbar
  - Kategorie deaktivieren: die Einfärbung ändert sich sofort
  - Farbe und Deckkraft ändern: wird sofort übernommen
  - `/zcm debug`: alle sichtbaren Zonen werden gelistet
  - Clientsprache wechseln: das Menü erscheint in der jeweiligen Sprache

## Nicht im Umfang (v1)

- Gamepad-UI
- Einfärbung auf Zonen- und Unterzonenkarten
- Prozentanzeige oder Teilfortschritt als Farbverlauf
- Persistenter Umriss-Cache über Sessions hinweg
