# Install

## 1. Copy the extension into place

The mod's folder name must match its content id, `sbe_station_trader`. Copy
the entire project folder there and rename it:

| OS | Extensions path |
|---|---|
| Linux (native Steam build) | `<Steam library>/steamapps/common/X4 Foundations/extensions/sbe_station_trader/` |
| Windows | `%USERPROFILE%\Documents\Egosoft\X4\extensions\sbe_station_trader\` |
| Steam Deck / Proton | `<compatdata>/pfx/drive_c/users/steamuser/Documents/Egosoft/X4/extensions/sbe_station_trader/` |

The folder must contain `content.xml` directly at its root — not nested one
level deeper:

```
extensions/
  sbe_station_trader/
    content.xml
    aiscripts/sbe_stationtrader.xml
    libraries/icons.xml
    t/0001.xml
```

On this machine the game is installed via Steam at
`~/.local/share/Steam/steamapps/common/X4 Foundations/`, and that's where
other installed extensions (DLCs, `tatertrader`, etc.) already live — so the
mod is installed there too, at
`~/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/`,
a straight copy of this repo's contents (minus `.git/`, which the game
doesn't need). If you edit the source in this repo, re-copy it over the
installed copy to pick up changes:

```bash
rsync -a --delete --exclude '.git' \
  ~/Projects/X4StationTrader/ \
  "~/.local/share/Steam/steamapps/common/X4 Foundations/extensions/sbe_station_trader/"
```

There is also a user-level extensions folder at
`~/.config/EgoSoft/X4/extensions/` that X4 scans in addition to the game
install directory — some modders prefer it because it isn't touched by
Steam's "verify integrity of game files." Only install the mod in **one**
of the two locations at a time; having the same content id (`sbe_station_trader`)
in both will make X4's extension manager report a duplicate-id conflict.

## 2. Enable it in-game

1. Launch X4: Foundations.
2. From the main menu, open **Extensions**.
3. Find **"Station Trader Assignments"** in the list and enable it.
4. Start or continue a game. No new-game requirement — it's a save-safe,
   non-breaking extension (`save="0"` in `content.xml`), so it can be added
   to or removed from an existing save.

If the extension doesn't appear in the list:
- Double-check the folder is directly under `extensions/` (not a subfolder
  of a subfolder — a common mistake when unzipping a downloaded copy).
- Confirm `content.xml` is valid XML: `xmllint --noout content.xml`.
- Check X4's own log for load errors — on Linux this is printed to the
  terminal you launched the game from, or captured in
  `~/.config/EgoSoft/X4/*.log` if logging is enabled in the game's advanced
  settings.

## 3. Verify the order shows up

1. Select any trade-capable ship you own (or one you can assign a
   commander/order to).
2. Open its order panel (default key: right-click the ship → **Orders**, or
   the order wheel in the ship's info menu).
3. Under the **Trade** category, look for **Station Trader**.
4. If it's there and its parameters (Home Station(s), Ware Priority List,
   etc.) render correctly, the extension loaded successfully.

See [CONFIGURATION.md](CONFIGURATION.md) for how to set it up.
