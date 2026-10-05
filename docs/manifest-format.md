# Gluon-Manifest: Format seit 2016

Stand 03.10.2026. Nachgelesen in freifunk-gluon/gluon (Tags, `scripts/`,
`Makefile`) und freifunk-gluon/packages (`admin/autoupdater`). Werkzeuge dazu in diesem Repo.

## Aufbau, unverändert seit 2016

```
BRANCH=stable
DATE=2026-10-03 04:19:45+02:00
PRIORITY=0

<image-zeilen>
---
<unterschriften, eine je Zeile, 128 Hex-Zeichen>
```

Unterschrieben wird alles vor `---`, gleich welches Zeilenformat (ecdsautils).

## Image-Zeilen nach Gluon-Version

| Gluon | erzeugte Zeilen je Modell | Quelle |
| --- | --- | --- |
| bis v2016.2.3 | `<modell> <version> <sha512> <datei>` | `Makefile` Ziel `manifest` (SHA512SUM) |
| v2016.2.4 bis v2017.1.x | zusätzlich davor `<modell> <version> <sha256> <datei>`, also sha256-Zeile, dann sha512-Zeile | Gluon `f9d59be7` (v2016.2.x) bzw. `9e6cfaee` (v2017.1), 25.02.2017 |
| v2018.1 bis v2020.2 | zuerst `<modell> <version> <sha256> <größe> <datei>`, danach die beiden alten Zeilen | `21b3dd32` "add file size field" (28.12.2017), `scripts/generate_manifest.sh` bzw. `.lua` |
| ab v2021.1 | nur noch `<modell> <version> <sha256> <größe> <datei>` | `2bfc39f3` "remove obsolete manifest lines (#2067)", 04.07.2020 |

Die Übergangszeit mit doppelten bzw. dreifachen Zeilen ging also von 2016.2.4
bis 2020.2.

## Was der Autoupdater des Knotens liest

| Autoupdater auf dem Knoten | liest | prüft |
| --- | --- | --- |
| Gluon 2016.2 (Lua, packages `4346733`) | 4-Feld-Zeilen; bei mehreren passenden die **letzte** | `sha512sum` |
| Gluon 2017.1 (Lua, packages `7182371`) | 4-Feld-Zeilen mit 64 Zeichen Prüfsumme | `sha256sum` |
| ab Gluon 2018.1 (C, `src/manifest.c`) | nur 5-Feld-Zeilen; 4-Feld-Zeilen werden übergangen | sha256 und Größe |

Daraus folgt die Reihenfolge, in der Gluon die Zeilen schrieb und
`manifest-altformat.sh` sie schreibt: 5-Feld-Zeile, 4-Feld-sha256, 4-Feld-sha512.
Ein 2016.2-Knoten findet dann als letzte passende Zeile die sha512-Zeile.

## Modellnamen

Der Knoten sucht seine Zeile über `platform_info.get_image_name()`. Die Namen
der ar71xx-Zeit (Gluon 2016.2 bis 2020.2) weichen teils von den heutigen ab,
zum Beispiel `tp-link-tl-wr1043n-nd-v4` (2016.2, `targets/ar71xx-generic/profiles.mk`)
gegenüber `tp-link-tl-wr1043nd-v4` heute. Gluon hat die alten Namen bis
v2023.2 als `manifest_aliases` mitgeführt ("upgrade from OpenWrt 19.07") und in
v2025.1 entfernt (`4e2bf620`, 09.02.2024). Die Paare aus v2023.2.6 (ergaenzt um Namen aus v2022.1/v2023.1 und der
4-Feld-Zeit) stehen als Array `ALIAS_ALT` in `manifest-altformat.sh` und
`manifeste-zusammenfuehren.sh`.

## Alte Knoten auf neue Firmware bringen

`manifest-altformat.sh` macht ein Manifest für alte Knoten lesbar. Allein
reicht das nicht:

- **Zwischenschritt.** Gluon 2025.1 aktualisiert nur ab v2022.1, und von
  ar71xx nach ath79 geht es nur mit passenden Images. Ein Knoten auf 2016.2
  braucht ein Manifest, das auf ein Zwischen-Image zeigt, das seine Firmware
  flashen kann. Welche Zwischenstufen nötig sind, ist je Gerät zu klären.
- **Unterschriften.** Das neue Manifest trägt keine Unterschrift mehr. Der
  alte Knoten zählt nur Schlüssel aus seiner eigenen (alten) site.conf und
  verlangt deren `good_signatures`. Signer, die nur noch dort stehen, müssen
  mit unterschreiben; die Unterschriften kommen hinter `---` dazu.
- **Spiegel.** Der Knoten fragt die Mirror-URLs aus seiner alten site.conf ab;
  das umgeschriebene Manifest muss dort liegen.
- **Version.** Die Version im Manifest muss für den alten Knoten neuer sein
  als seine eigene (alter Lua-Vergleich in `autoupdater/version.lua`).

## Werkzeug

```bash
manifest-altformat.sh -o /tmp/stable.manifest \
    /var/www/firmware/stable/21_dias/sysupgrade/stable.manifest
```

Prüft vorher je Image die sha256 gegen das Manifest, berechnet die sha512,
verwirft vorhandene 4-Feld-Zeilen und alle Unterschriften und schreibt je
Modell (und je altem Namen mit `-a`) die drei Zeilen. Getestet am
nachgestellten Manifest mit nachgebauter Auswahllogik der drei
Autoupdater-Generationen: alle drei finden ihre Zeile und die passende
Prüfsumme. Am echten Knoten nicht erprobt.
