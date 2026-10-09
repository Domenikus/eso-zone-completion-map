# Zone Completion Map

An Elder Scrolls Online addon that tints every zone you have fully completed on the
Tamriel and Aurbis world maps, so you can see at a glance where work is left.

A zone counts as complete when every enabled Zone Guide category that has activities in
that zone is done. That includes the zone story (priority quests), wayshrines, delves,
skyshards, world bosses and so on. You can enable or disable each category in the settings.

## Requirements

- [LibAddonMenu-2.0](https://www.esoui.com/downloads/info7-LibAddonMenu.html)

## Installation

Copy the `ZoneCompletionMap` folder into
`Documents/Elder Scrolls Online/live/AddOns/`.

## Usage

- Settings: *Settings → Addons → Zone Completion Map*. Here you can change the
  enabled state, tint color and opacity, the categories that count, and **Invert tint**,
  which tints the zones that are not yet complete instead. Zones without any tracked
  activity stay untinted in both modes.
- `/zcm debug`: with the Tamriel or Aurbis map open, lists every detected zone in chat
  and whether it counts as complete.
- `/zcm raw [filter]`: diagnostic output of the raw map data per zone outline, including
  outlines the addon cannot use. The optional filter matches the zone name.

## Translations

The addon supports English, German, French, Spanish, Russian, Japanese and Simplified
Chinese. Category names come from the game itself. The addon's own texts were
machine-translated, so corrections from native speakers are welcome.

## In-game test checklist

- [ ] Tamriel map: completed zones are green and incomplete zones are not.
- [ ] Aurbis map: Coldharbour, Clockwork City and the other realms are tinted correctly.
- [ ] Zoom and pan: the tint stays aligned with the zone outlines.
- [ ] Hovering over a tinted zone still shows the normal highlight.
- [ ] Opening a zone map and then going back to Tamriel: no tint remains on the zone map.
- [ ] Disabling a category updates the tint immediately.
- [ ] Changing color or opacity is applied immediately.
- [ ] `/zcm debug` lists all visible zones.
- [ ] Switching the client language shows the settings menu in that language.
