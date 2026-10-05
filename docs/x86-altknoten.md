# HowTo: alte x86-Knoten auf Gluon 2025.1 bringen

Stand 05.10.2026. Für Betreiber eines Firmware-Servers, die x86-Knoten mit
Gluon bis 2021.1 per Autoupdater auf 2025.1 holen wollen, ohne dass jemand
vor Ort etwas einstellen muss (BIOS, Boot-Optionen, Config-Mode).

## Kurzfassung

1. Release-Verzeichnis: Zu jedem x86-sysupgrade-Image liegt in `other/` ein
   `...-<target>-mbr-sysupgrade.img.gz`. Neanderfunk baut es seit
   gluon-patches-hardware `a50d4b3` (FirmwareConfigs v2025.1.x `402e37a`).
2. Manifest so erzeugen, dass alte x86-Knoten dieses MBR-Image bekommen und
   alle anderen das normale EFI-Image:
   `manifeste-zusammenfuehren.sh` oder `manifest-altformat.sh`, beide mit
   `-x alt` (Vorgabe) bzw. `-x alle`.
3. Prüfen, unterschreiben, freischalten wie immer.

Danach läuft der Knoten mit 2025.1 und seiner alten Konfiguration. Beim
nächsten Release wechselt er von selbst auf das EFI-Image.

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

| Gluon auf dem Knoten | OpenWrt | Boot-Partition | Manifest-Zeilen, die er liest | mit MBR-Image |
| --- | --- | --- | --- | --- |
| bis 2016.2.3 | 14.07 / 15.05 | 4 MB | 4 Felder, sha512 (letzte passende Zeile) | **Konfiguration geht trotzdem verloren** |
| 2016.2.4 bis 2016.2.x | 15.05 | 4 MB | 4 Felder, sha256 | **Konfiguration geht trotzdem verloren** |
| 2017.1.x | 17.01 | 16 MB | 4 Felder, sha256 | bleibt erhalten |
| 2018.1 bis 2021.1.x | 17.01 bis 19.07 | 16 MB | 5 Felder | bleibt erhalten |
| ab 2022.1 | ab 21.02 | 16 MB | 5 Felder | nicht nötig (EFI-Image geht) |

**Grenze bis Gluon 2016.2 (Barrier Breaker, Chaos Calmer):** Deren
Bootpartition ist 4 MB groß; OpenWrt hat sie erst zu 17.01 auf 16 MB gebracht
(`12a6e3cd054`, 09.11.2016). Das alte sysupgrade liest nach dem `dd` die
Partitionstabelle nicht neu ein, `/dev/sda1` hat für den Kernel also noch
4 MB, das neue Dateisystem ist 16 MB groß: `EXT4-fs (sda1): bad geometry:
block count 4096 exceeds size of device (1024 blocks)`, `mount ... Invalid
argument`, die Konfiguration landet im RAM (im Labortest so gesehen). Ein
Image mit 4-MB-Boot geht nicht, allein der 2025.1-Kernel hat 6 MB. Solche
Knoten kommen ohne Konfiguration hoch: ohne Schlüssel-Image im Setup-Mode
(aus dem Mesh, braucht jemanden vor Ort), mit Schlüssel-Image im
Normalbetrieb mit Vorgaben (ohne Kontakt, Standort, Hostname).

Modellnamen: `x86-generic`, `x86-64` und die alten Varianten `x86-kvm`,
`x86-virtualbox`, `x86-vmware`, `x86-xen_domu`, `x86-64-virtualbox`,
`x86-64-vmware` (stehen als Aliase in den Manifest-Werkzeugen). `x86-geode`
ist nicht betroffen.

Ohne Weg per Autoupdater:

- Gluon 2014.x auf x86: `platform_info.get_image_name()` liefert dort `nil`,
  der Autoupdater bricht mit "doesn't support this hardware model" ab (im
  2014.4-Rootfs des VDI nachgesehen). Kein Manifest hilft; nur von Hand mit
  dem MBR-Image flashen. `x86-generic` als Modellname gibt es ab 2015.1.
- `x86-xen` (Ziel x86-xen_domu, bis 2016.2): kein heutiges Image, Knoten
  meldet "No matching firmware found" und bleibt stehen. Von Hand umstellen.

Ein Alias-Name hilft nicht: Alte und neue x86-Knoten melden denselben
Modellnamen. Unterscheiden lässt sich nur über das Zeilenformat (4 Felder =
bis 2017.1) oder darüber, welches Manifest der Knoten liest (eigene
Mirror-URL, eigener Zweig).

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

Typischer Fall: unser stable, das heutige Knoten und eingesammelte Altknoten
gemeinsam lesen. Vorgabe `-x alt`: nur die 4-Feld-Zeilen zeigen aufs
MBR-Image, die 5-Feld-Zeilen bleiben beim EFI-Image.

```sh
manifeste-zusammenfuehren.sh -n -b <basis> -z <zusatz> -o <neu>
```

Im Probelauf je Domain auf die x86-Zeilen achten:

```
  x86: 2 Image(s) auf MBR umgelenkt (nur 4-Feld-Zeilen)
```

Fehlt ein MBR-Image, bricht das Skript ab und nennt die Datei. Mit `-x aus`
lässt es sich bewusst übergehen (dann verlieren alte x86-Knoten ihre
Konfiguration). Danach ohne `-n` laufen lassen.

Grenze: Knoten mit 2018.1 bis 2021.1 lesen dieselben 5-Feld-Zeilen wie
heutige Knoten und bekommen hier das EFI-Image. Gibt es solche x86-Knoten
(Karte: Firmware-Version und Modell), braucht es Fall 2b für sie.

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
Werkzeuge die 4-Feld-Zeilen von `x86-generic` und seinen Aliasen darauf. Der
Knoten bleibt danach auf x86-legacy, etwas langsamer, aber er startet.
Ohne x86-legacy-Image melden die Werkzeuge das als Warnung. Neanderfunk baut
x86-legacy seit FirmwareConfigs v2025.1.x `9706eab` (05.10.2026).

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
- Die Übernahme aus 2017.1 bis 2021.1 ist nicht am echten Altimage getestet
  (keins im Archiv); Geometrie identisch (Start Sektor 512, 16 MB), die
  Konfigurationsübernahme selbst ist mit simulierter Übergabe geprüft.
