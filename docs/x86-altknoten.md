# HowTo: alte x86-Knoten auf Gluon 2025.1 bringen

Stand 06.10.2026. Für Betreiber eines Firmware-Servers, die x86-Knoten mit
Gluon bis 2021.1 per Autoupdater auf 2025.1 holen wollen, ohne dass jemand
vor Ort etwas einstellen muss (BIOS, Boot-Optionen, Config-Mode).

## Kurzfassung

> **x86 vor Gluon 2016.2.6 scheitert auf jeden Fall.** Ein direkter Sprung
> auf irgendeine Firmware ab LEDE 17.01 (Gluon 2017.1) verliert dort die
> Konfiguration, egal welches Image (Gluon #1010). Weg nur über den
> Zwischenschritt Gluon 2016.2.6+ oder von Hand, siehe Sprungmatrix.

1. Release-Verzeichnis: Zu jedem x86-sysupgrade-Image liegt in `other/` ein
   `...-<target>-mbr-sysupgrade.img.gz` (MBR, Bootpartition ext4).
   Neanderfunk baut es seit gluon-patches-hardware `a50d4b3`.
2. Manifest mit `manifeste-zusammenfuehren.sh` oder `manifest-altformat.sh`.
   Für x86 (generic, legacy, 64) schreiben beide: keine sha512-4-Feld-Zeile
   (die lesen nur Knoten bis 2016.2.3), die sha256-4-Feld-Zeile aufs
   MBR-Image, die 5-Feld-Zeile wie die Basis (EFI). Mit `-x alle` auch die
   5-Feld-Zeile aufs MBR-Image, nur für Verzeichnisse, die allein Altknoten
   lesen.
3. Prüfen, unterschreiben, freischalten wie immer.

Danach läuft der Knoten mit 2025.1 und seiner alten Konfiguration. Beim
nächsten Release wechselt er von selbst auf das EFI-Image (getestet).

## Warum ein eigenes Image

Gluon baut x86 seit 2023.2 als `-squashfs-combined-efi`: GPT, Partition 1 ist
eine FAT-Partition (EFI-Systempartition), Partition 2 das Rootfs. OpenWrt bis
19.07, also Gluon bis 2021.1, legt beim sysupgrade die gesicherte
Konfiguration so ab:

```sh
mount -t ext4 -o rw,noatime "/dev/$partdev" /mnt
cp -af "$CONF_TAR" /mnt/
```

Auf FAT schlägt das `mount` fehl, `sysupgrade.tgz` landet im RAM, und der
Knoten startet ohne Konfiguration im Setup-Mode. Im Labortest mit einem
echten Gluon-2014.4-Knoten (VirtualBox-Image, QEMU) nachgewiesen:

```
mount: mounting /dev/sda1 on /mnt failed: Invalid argument
Upgrade completed
```

Upstream ist das Gluon #2967. Gluon 2023.1.1 hat EFI dafür wieder entfernt,
2023.2 hat es zurückgebracht und setzt als Untergrenze 2022.1. Erst OpenWrt
21.02 erkennt FAT (`part_magic_fat`).

OpenWrt baut das alte Format `-squashfs-combined` (MBR, Partition 1 ext4)
weiterhin (`CONFIG_GRUB_IMAGES` ist Vorgabe), Gluon kopiert es nur nicht
heraus. Der Patch `targets/x86-mbr-sysupgrade.sh` in gluon-patches-hardware
legt es als Zusatz-Image nach `images/other/`. Für x86-geode ist das unnötig:
Gluon baut dort ohnehin nur `-squashfs-combined`.

## Welche Knoten betroffen sind

| Gluon auf dem Knoten | OpenWrt | Boot | liest im Manifest | Werkzeuge liefern für x86 | Ergebnis |
| --- | --- | --- | --- | --- | --- |
| 2014.x | 14.07 | 4 MB | (Autoupdater kennt x86 nicht) | - | nur von Hand |
| 2015.1 bis 2016.2.3 | 14.07 / 15.05 | 4 MB | 4 Felder, sha512 | nichts | Knoten bleibt stehen |
| 2016.2.4 bis 2016.2.5 | 15.05 | 4 MB | 4 Felder, sha256 | MBR-Image | **Konfiguration verloren** |
| 2016.2.6 bis 2016.2.7 | 15.05 | 4 MB | 4 Felder, sha256 | MBR-Image | bleibt erhalten (gestuftes sysupgrade) |
| 2017.1.x | 17.01 | 16 MB | 4 Felder, sha256 | MBR-Image | bleibt erhalten |
| 2018.1 bis 2021.1.x | 17.01 bis 19.07 | 16 MB | 5 Felder | EFI-Image (`-x alle`: MBR) | EFI: **verloren**; MBR: bleibt erhalten |
| ab 2022.1 | ab 21.02 | 16 MB | 5 Felder | EFI-Image | bleibt erhalten |

2016.2.4/2016.2.5 und 2016.2.6+ lesen dieselben Zeilen und lassen sich im
Manifest nicht trennen; die Werkzeuge entscheiden sich für den Weg, der
2016.2.6+ und 2017.1 trägt. Knoten auf 2016.2.4/2016.2.5 vorher auf 2016.2.6+
bringen. Ein x86-Knoten ohne Konfiguration kommt ohne Schlüssel-Image im
Setup-Mode hoch (aus dem Mesh, braucht jemanden vor Ort), mit Schlüssel-Image
im Normalbetrieb mit Vorgaben (ohne Kontakt, Standort, Hostname).

### Warum vor 2016.2.6 nichts geht

Die Bootpartition war bis Chaos Calmer 4 MB groß; OpenWrt hat sie zu 17.01
auf 16 MB gebracht (`12a6e3cd054`, 09.11.2016). Das alte sysupgrade liest
nach dem `dd` die Partitionstabelle nicht neu ein, `/dev/sda1` hat für den
Kernel also noch 4 MB, das neue Dateisystem 16 MB: `EXT4-fs (sda1): bad
geometry: block count 4096 exceeds size of device (1024 blocks)`,
`mount ... Invalid argument`, die Konfiguration landet im RAM (im Labortest
so gesehen). Gluon hat das in **v2016.2.6** mit dem gestuften sysupgrade
behoben (`d4a69c00`, Backport aus LEDE; Release Notes 2016.2.6 und 2017.1:
"a Gluon node running an older version must be upgraded to Gluon v2016.2.6
first before switching to a LEDE-based version"). Ein Image mit 4-MB-Boot
geht nicht, allein der 2025.1-Kernel hat 6 MB.

## Sprungmatrix (x86, Konfiguration bleibt erhalten)

32 Bit jeweils auf x86-legacy, siehe "32 Bit".

| Herkunft | Weg | Sprünge |
| --- | --- | --- |
| 2014.x | kein Autoupdater (Image-Name `nil`): von Hand Sicherung ziehen (`sysupgrade -b`), 2025.1 flashen, Sicherung als `/boot/sysupgrade.tgz` auf Partition 1 legen, `firstboot -y; reboot` (wird eingespielt wie nach sysupgrade) | 1 von Hand |
| 2015.1 bis 2016.2.5 | -> Gluon 2016.2.6/2016.2.7 (gleiche 4-MB-Aufteilung) -> 2025.1-MBR -> später EFI | 2 (+1 automatisch) |
| 2016.2.6 bis 2016.2.7 | -> 2025.1-MBR -> später EFI | 1 (+1) |
| 2017.1 | -> 2025.1-MBR -> später EFI | 1 (+1) |
| 2018.1 bis 2021.1 | -> 2025.1-MBR (`-x alle`, eigenes Verzeichnis) -> später EFI | 1 (+1) |
| ab 2022.1 | -> 2025.1-EFI | 1 |

Die Zwischenstufe 2016.2.6+ ist ein eigener Bau von Gluon v2016.2.7 für
x86-generic (Chaos-Calmer-Zeit, i486) mit der Site der Zielcommunity; die
Werkzeuge lenken die sha512-Zeilen nicht darauf, das Manifest dafür ist von
Hand zu bauen. Getestet: 2014.4 -> MBR (Konfiguration verloren, Ursache
nachgewiesen), MBR -> EFI (erhalten). Nicht getestet: 2016.2.6+ -> MBR und
2017.1 bis 2021.1 -> MBR (Geometrie wie beim Upstream-Sprung auf 2017.1:
ext4, 16 MB, Start Sektor 512).

Modellnamen: `x86-generic`, `x86-64` und die alten Varianten `x86-kvm`,
`x86-virtualbox`, `x86-vmware`, `x86-xen_domu`, `x86-64-virtualbox`,
`x86-64-vmware` (Aliase in den Werkzeugen). `x86-geode` gibt es erst ab
2017.1 (16-MB-Boot) und ist nicht betroffen.

Ohne Weg per Autoupdater:

- Gluon 2014.x auf x86: `platform_info.get_image_name()` liefert dort `nil`,
  der Autoupdater bricht mit "doesn't support this hardware model" ab (im
  2014.4-Rootfs nachgesehen). Nur von Hand, siehe Sprungmatrix.
- `x86-xen` (Ziel x86-xen_domu, bis 2016.2): kein heutiges Image, Knoten
  meldet "No matching firmware found" und bleibt stehen. Von Hand umstellen.

Ein Alias-Name hilft nicht: Alte und neue x86-Knoten melden denselben
Modellnamen. Unterscheiden lässt sich nur über das Zeilenformat oder darüber,
welches Manifest der Knoten liest (eigene Mirror-URL, eigener Zweig).

## Wie alt sind x86-Knoten im Feld noch? (Stand 06.10.2026)

Ausgewertet: alle Karten, die der Gluon-Census
([census-exporter](https://github.com/freifunk-gluon/census-exporter),
`communities.json`) abfragt, 122 von 130 erreichbar, 37.941 Knoten. x86
erkannt am Image-Namen oder am Modell (CPU- bzw. DMI-Name, keine
Router-Hersteller), auch offline gemeldete Knoten gezählt.

| Gluon auf x86 | Knoten | Bedeutung |
| --- | --- | --- |
| 2014.x, 2015.x | **0** | der Handweg bleibt Theorie |
| 2016.1.x | 1 | vor 2016.2.6: Zwischenschritt nötig; CPU der Athlon-XP-Klasse ohne SSE2, also x86-legacy |
| 2016.2.0 bis 2016.2.5 | 1 (2016.2.4, VM) | vor 2016.2.6: Zwischenschritt nötig |
| 2016.2.6 bis 2016.2.7 | 2 | direkt aufs MBR-Image |
| 2017.1.x | 10 | direkt aufs MBR-Image |
| 2018.1 bis 2021.1 | 177 | MBR-Image über `-x alle` |
| ab 2022.1 | 1.326 | EFI-Image geht |

Die älteste x86-Firmware im Feld ist also Gluon **2016.1**; uralte x86-VMs
(2014/2015) gibt es nach dem Census nicht mehr. Nicht erfasst: Communities
ohne Census-Eintrag und die 8 Karten, die beim Abruf nicht antworteten.

## Schritte auf dem Firmware-Server

Werkzeuge: dieses Repo, siehe README.

### 1. Images prüfen

Je Domain müssen neben dem EFI-Image die MBR-Images liegen:

```sh
ls <release>/<domain>/other/ | grep mbr-sysupgrade
```

Erwartet: `gluon-<site>-<release>-x86-generic-mbr-sysupgrade.img.gz` und
`...-x86-64-mbr-sysupgrade.img.gz`, mit x86-legacy im Bau auch
`...-x86-legacy-mbr-sysupgrade.img.gz` (siehe "32 Bit" unten).

### 2a. Gemeinsames Verzeichnis (alte und heutige Knoten lesen dasselbe Manifest)

Typischer Fall: stable, das heutige Knoten und eingesammelte Altknoten
gemeinsam lesen. Vorgabe: x86-5-Feld-Zeilen bleiben beim EFI-Image, die
sha256-4-Feld-Zeile zeigt aufs MBR-Image, keine sha512-Zeile.

```sh
manifeste-zusammenfuehren.sh -n -b <basis> -z <zusatz> -o <neu>
```

Im Probelauf je Domain:

```
  x86: 2 Image(s) auf MBR umgelenkt (nur sha256-4-Feld-Zeilen; keine sha512-Zeilen fuer x86)
```

Fehlt ein MBR-Image, schreibt das Skript für x86 gar keine 4-Feld-Zeilen und
warnt; alte x86-Knoten bleiben dann stehen. Danach ohne `-n` laufen lassen.

Grenze: Knoten mit 2018.1 bis 2021.1 lesen dieselben 5-Feld-Zeilen wie
heutige Knoten und bekommen hier das EFI-Image (Konfiguration verloren). Gibt
es solche x86-Knoten (Karte: Firmware-Version und Modell), braucht es Fall 2b.

### 2b. Eigenes Verzeichnis nur für Altknoten

Für Knoten, die über eine eigene Mirror-URL oder einen eigenen Zweig kommen
(z. B. die übernommene Community auf ihren alten Pfaden): alle x86-Zeilen
aufs MBR-Image.

```sh
manifeste-zusammenfuehren.sh -x alle -b <basis> -z <zusatz> -o <altknoten>
```

Oder ohne Zusammenführen für ein einzelnes Manifest:

```sh
cd <altknoten>/<domain>/sysupgrade
ln -s ../other/gluon-<site>-<release>-x86-64-mbr-sysupgrade.img.gz .
ln -s ../other/gluon-<site>-<release>-x86-generic-mbr-sysupgrade.img.gz .
manifest-altformat.sh -x alle -o stable.manifest.neu stable.manifest
```

`manifest-altformat.sh` nennt fehlende Symlinks mit dem passenden
`ln`-Befehl. Heutige Knoten dürfen dieses Manifest nicht lesen: Ein Rechner,
der nur per UEFI bootet, startet kein MBR-Image.

### 3. Prüfen, unterschreiben, freischalten

```sh
manifest-pruefen.sh <neu>
```

Dann unterschreiben (ecdsasign bzw. Gluons `contrib/sign.sh`, auch mit
Schlüsseln, die nur noch alte Knoten kennen), zuletzt den Symlink (z. B.
`stable`) umstellen.

Gegenprobe, was ein alter Knoten zieht:

```sh
grep '^x86-generic ' <neu>/<domain>/sysupgrade/stable.manifest
```

Die Zeilen mit 4 Feldern müssen auf `...-mbr-sysupgrade.img.gz` zeigen.

## Was am Knoten passiert

1. Der alte Autoupdater lädt das MBR-Image, das alte sysupgrade schreibt es
   per `dd` auf die Platte und legt `sysupgrade.tgz` auf die
   ext4-Partition 1. Das alte sysupgrade prüft auf x86 die Boot-Signatur
   (`eb48`/`eb63`), aber keine compat-version und keine Geräte-Metadaten
   (bis 19.07 nachgesehen: `REQUIRE_IMAGE_METADATA` ist für x86 nicht
   gesetzt, fwtool kennt noch keine compat-version).
2. 2025.1 startet, holt die Konfiguration aus Partition 1 und führt die
   Gluon-Upgrade-Skripte aus. Der Migrationsassistent
   (`neanderfunk-legacy-migrate`, ab Feed `fe80e15`) übernimmt die alte
   Konfiguration: Rollen der Karten nach der alten Aufzählung, weitere
   Karten, `primary_mac`, VPN-Limit, Branch usw.
3. Der Knoten liest jetzt die Mirrors aus der 2025.1-site.conf. Bei gleicher
   Version passiert nichts weiter.
4. Beim nächsten Release kommt das normale EFI-Image. OpenWrt 24.10 erkennt
   das andere Partitionslayout, schreibt das ganze Image und legt die
   Konfiguration auf die FAT-Partition (erkennt FAT). Der Rechner bootet
   weiter wie bisher: Das Gluon-EFI-Image hat zusätzlich eine
   BIOS-Boot-Partition und startet auch im BIOS-/CSM-Modus (in QEMU mit
   SeaBIOS gesehen). Ein Knoten, der vorher ein altes Gluon gebootet hat,
   bootet zwangsläufig im BIOS-/CSM-Modus, denn ältere Images waren reine
   MBR-Images. Im BIOS muss also niemand etwas umstellen.

Ob das MBR-Image ewig mitgebaut wird, ist egal: Solange alte Knoten
eingesammelt werden, kostet es nur Speicherplatz auf dem Server (rund 24 MB je
Target und Domain).

## 32 Bit: CPU-Grundlage

OpenWrt x86/generic verlangt seit LEDE 17.01 einen Pentium 4 (SSE2, OpenWrt
`febc6cc5bf`). Bis Gluon 2016.2 war "generic" i486-Klasse. Ein Knoten bis
2016.2 auf einer CPU ohne SSE2 (Pentium III, Athlon XP, VIA C3 u. a.) startet
das heutige x86-generic-Image nicht. x86-legacy (i486) läuft dagegen auf jeder
32-Bit-CPU.

Liegt ein `x86-legacy-mbr-sysupgrade.img.gz` in der Basis, lenken beide
Werkzeuge die sha256-4-Feld-Zeile von `x86-generic` und seinen Aliasen darauf
(2016.2.6+ ist noch i486-Klasse). Der Knoten bleibt danach auf x86-legacy,
etwas langsamer, aber er startet. Ohne x86-legacy-Image melden die Werkzeuge
das als Warnung. Neanderfunk baut x86-legacy seit FirmwareConfigs v2025.1.x
`9706eab`.

## Labortest (06.10.2026)

QEMU, echter Gluon-2014.4-Knoten (Barrier Breaker, drei Karten), Image
26100521bro (x86-64; x86-generic und x86-legacy waren nicht im Lauf, gleicher
Aufbau):

- sysupgrade 2014.4 -> MBR-Image: Image geschrieben, bootet; Konfiguration
  verloren (4-MB-Grenze oben, nachgewiesen).
- danach sysupgrade MBR -> EFI-Image (wie beim nächsten Release): "Partition
  layout has changed. Full image will be written", Konfiguration erhalten
  (Hostname, Kontakt, VPN-Limit), `/boot` jetzt vfat, bootet im BIOS-Modus.
- 64-MB-Platte: `dd` endet mit "No space left on device", Kernel kürzt die
  Rootfs-Partition ("extends beyond EOD, truncated"), Knoten bootet mit
  27,5 MB Overlay. Unkritisch.
- Echte Gluon-2021.1-Images (x86-64, x86-generic), drei Karten, eingerichteter
  Knoten: sysupgrade auf das 2025.1-MBR-Image (x86-64 bzw. für x86-generic
  das x86-legacy-MBR-Image) behält Hostname, Kontakt, Koordinaten, VPN mit
  Limit, alle drei Kartenrollen und die node_id. Gegenprobe auf das
  EFI-Image: Konfiguration weg. 2017.1 bis 2020.2 nicht eigens getestet
  (gleiche Geometrie: Start Sektor 512, 16 MB).
- 2014.4 -> 2021.1 per echtem sysupgrade: Konfiguration weg (4-MB-Grenze);
  von Hand (Sicherung als `sysupgrade.tgz` in Partition 1) vollständig
  übernommen, danach weiter auf 2025.1-MBR ohne Verlust.
- 32-Bit-MBR-Images aus 26100600bro, QEMU i386 im BIOS-Modus: beide mit
  Partitionstabelle wie x86-64 (ext4-Boot ab Sektor 512, 16 MB).
  x86-legacy bootet auf `-cpu pentium` (ohne SSE2) bis in den Betrieb (Setup,
  Neustart, batman an eth0). x86-generic bootet auf `-cpu n270` (Atom, SSE2);
  auf `-cpu pentium3` stirbt procd sofort ("invalid opcode" in libubox, Kernel
  panic). Das bestätigt die Umlenkung von x86-generic auf x86-legacy oben.
