# gluon-manifest-tools

Werkzeuge für den Firmware-Server einer Gluon-Community: Manifeste so
erzeugen und prüfen, dass auch Knoten mit sehr alter Firmware (Gluon 2014.x
bis 2021.1) ihr Update finden. Die Skripte brauchen nur bash, awk, GNU
coreutils und sha256sum/sha512sum; sie lesen die übliche Ablage
`<wurzel>/<domain>/{sysupgrade,factory,other}` oder, bei nur einer Domain,
flach `<wurzel>/{sysupgrade,factory,other}`, und schreiben unsignierte
Manifeste. Hilfe jeweils mit `--help`.

| Skript | Zweck |
| --- | --- |
| `manifeste-zusammenfuehren.sh` | Neues Firmware-Verzeichnis (z. B. stable) aus einer Basis (aktuelle Gluon-Version) und einem Zusatz (letzte Version für Geräte, die die Basis nicht mehr baut), nur per Symlinks. Neues Manifest mit allen Zeilenformaten, die alte Knoten lesen, alten Modellnamen (eingebaute Aliasliste) und x86-Umlenkung auf ein MBR-Image (`-x`). `-g` prüft die Abdeckung gegen einen Gluon-Quellbaum. |
| `manifest-altformat.sh` | Ein vorhandenes Manifest um die alten Zeilenformate und alten Modellnamen ergänzen. |
| `manifest-pruefen.sh` | Manifeste gegen die Images daneben prüfen (Datei da, sha256, Größe), vor und nach dem Unterschreiben. |

Unterschreiben gehört nicht dazu: dafür hat jede Community `ecdsasign` bzw.
Gluons `contrib/sign.sh`. Die neuen Manifeste tragen keine Unterschrift.

> **x86 vor Gluon 2016.2.6 scheitert auf jeden Fall:** Beim Sprung auf eine
> Firmware ab LEDE 17.01 (Gluon 2017.1) geht die Konfiguration verloren, egal
> welches Image (Bootpartition 4 -> 16 MB, Gluon #1010). Weg nur über den
> Zwischenschritt Gluon 2016.2.6+ oder von Hand, siehe Sprungmatrix unten.

## Welche Firmware welche Zeilen liest

| Gluon auf dem Knoten | liest |
| --- | --- |
| bis 2016.2.3 | `<modell> <version> <sha512> <datei>`, die letzte passende |
| 2016.2.4 bis 2017.1 | `<modell> <version> <sha256> <datei>` |
| ab 2018.1 | `<modell> <version> <sha256> <größe> <datei>` |

Gluon selbst schreibt seit 2021.1 nur noch die letzte Form. Einzelheiten,
Quellen und Grenzen: [docs/manifest-format.md](docs/manifest-format.md).

## Fälle ohne Weg per Autoupdater

- TP-Link CPE210/220/510/520 v1 auf Gluon 2016.2: deren sysupgrade lehnt das
  ath79-Image ab, nachdem das Netz schon gestoppt ist (Knoten offline bis
  Stromreset). Die Skripte schreiben für sie keine 4-Feld-Zeilen.
- x86 bis Gluon 2016.2.5: siehe oben; die Werkzeuge schreiben für x86 keine
  sha512-Zeilen (die lesen nur Knoten bis 2016.2.3), diese Knoten bleiben
  stehen.
- x86 mit Gluon bis 2021.1 auf ein EFI-Image (Gluon ab 2023.2): Konfiguration
  geht verloren. Weg über ein zusätzliches MBR-Image (sha256-Zeilen
  automatisch, 5-Feld-Zeilen mit `-x alle`), siehe
  [docs/x86-altknoten.md](docs/x86-altknoten.md).
- x86 mit Gluon 2014.x: der Autoupdater kennt dort keinen Image-Namen, nur
  von Hand.
- `x86-xen` (bis 2016.2): kein heutiges Image.
- Netgear WNDR3700 v4: der Wechsel ar71xx -> ath79 wurde nie gegangen
  (Kernelpartition gewachsen), bewusst kein Alias.

Ubiquiti NanoStation (loco) M XW sind per Alias enthalten: der Sprung von
Gluon 2021.1 (ar71xx) auf ein ath79-Image ist an einem Gerät im Feld ohne
Eingriff durchgelaufen.

## Sprungmatrix x86 (Konfiguration bleibt erhalten)

| Herkunft | Boot alt | Weg | Sprünge |
| --- | --- | --- | --- |
| 2014.x | 4 MB | von Hand: `sysupgrade -b`, neue Firmware flashen, Sicherung als `/boot/sysupgrade.tgz`, `firstboot -y; reboot` | 1 von Hand |
| 2015.1 bis 2016.2.5 | 4 MB | -> Gluon 2016.2.6/2016.2.7 -> 2025.1-MBR -> später EFI | 2 (+1 automatisch) |
| 2016.2.6 bis 2016.2.7 | 4 MB | -> 2025.1-MBR -> später EFI | 1 (+1) |
| 2017.1 | 16 MB | -> 2025.1-MBR -> später EFI | 1 (+1) |
| 2018.1 bis 2021.1 | 16 MB | -> 2025.1-MBR (`-x alle`, eigenes Verzeichnis) -> später EFI | 1 (+1) |
| ab 2022.1 | 16 MB | -> 2025.1-EFI | 1 |

Im Feld ist die älteste x86-Firmware nach dem Gluon-Census (06.10.2026)
Gluon 2016.1 (1 Knoten); vor 2016.2.6 sind es 2, 2014.x/2015.x keiner.

Was die Werkzeuge je Zeilenformat für x86 schreiben und was getestet ist:
[docs/x86-altknoten.md](docs/x86-altknoten.md).

## Beispiel

```sh
manifeste-zusammenfuehren.sh -n \
    -b /var/www/firmware/stable-v2025.1 \
    -z /var/www/firmware/sackgasse-v2021.1 -M sackgasse.manifest \
    -o /var/www/firmware/stable-neu
manifeste-zusammenfuehren.sh -b ... -z ... -o /var/www/firmware/stable-neu
manifest-pruefen.sh /var/www/firmware/stable-neu
```

Erst mit `-n` (Probelauf), dann unterschreiben, dann den Symlink des Zweigs
umstellen.

## Herkunft

Entstanden bei Freifunk im Neanderland (Neanderfunk) für die Übernahme alter
Knoten auf Gluon 2025.1 und die letzte Gluon-2021.1-Version für 4/32-Geräte.
Passend dazu für die Konfiguration der alten Knoten: das Paket
`neanderfunk-legacy-migrate` in
[Neanderfunk/packages](https://github.com/Neanderfunk/packages).

Lizenz: BSD-3-Clause, siehe [LICENSE](LICENSE).
