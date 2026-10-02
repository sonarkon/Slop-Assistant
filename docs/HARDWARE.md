# Onkyo-Bridge — Hardware-Referenz

Zentrale Übersicht über Pinbelegung, Beschaltung und Stückliste. Board ist
ein **ESP32-S3-DevKitC-1** (WROOM-1 N16R8, 16 MB Flash, 8 MB Octal-PSRAM).

Firmware-seitig gepflegt in
`airplay-esp32/config/sdkconfig.user.onkyo` — bei Abweichungen gilt die
Config als Quelle der Wahrheit, dieses Dokument fasst nur zusammen.

## Pinbelegung

| GPIO | Funktion | Beschaltung |
|---|---|---|
| **4** | RI-Sender | 470 Ω → Klinke Tip · GND → Sleeve |
| **5** | IR-Sender (Transistortreiber) | siehe unten |
| **6** | OLED SDA | I²C zum SSD1306 |
| **7** | OLED SCL | I²C zum SSD1306 |
| **12** | Koax-Ausgang (S/PDIF) | siehe unten |

Stand 09.09.2026: IR-Sender auf GPIO5 statt 6, OLED I²C auf GPIO6/7 statt
7/15 — beim Löten ist die Verdrahtung um einen Pin verrutscht, die
Konfiguration folgt der tatsächlichen Verdrahtung statt umgekehrt.

**Frei/nicht mehr verkabelt:** 11 und 13 (DAC-Reste — entfallen mit dem
Umstieg auf S/PDIF), 15, 16, 17, 18, 21.

**Gesperrt beim N16R8, nicht benutzen:** 33–37 (Octal-PSRAM), 26–32
(SPI-Flash), 19/20 (natives USB), 43/44 (UART0), 0/3/45/46 (Strapping-Pins),
48 (RGB-Status-LED des Boards).

## RI-Sender (GPIO4)

```
GPIO4 ──[470 Ω]── Klinke Tip (3,5 mm mono)
GND   ──────────── Klinke Sleeve
```

Reiner Gleichspannungspuls, kein Träger. 12-Bit-Protokoll, MSB first —
vollständig dokumentiert in `RI-CODES.md`. Der 470-Ω-Widerstand ist Dauerschutz
gegen die ESD-Klemmdiode des GPIO, kein Level-Shifting: der RI-Bus liegt im
Ruhezustand auf Masse (gemessen, kein Pull-up), 3,3 V reichen dem R-1045 als
High.

## IR-Sender (GPIO5) — Transistortreiber

```
5V ──[33 Ω]──►│── LED (Anode oben) ──┐
                                     │ Kollektor
GPIO5 ──[1,2 kΩ]──┤ Basis             │  NPN (BC337 oder 2N2222)
                   └─ Emitter ────────┴── GND
```

Rund 100 mA Pulsstrom (33 Ω an 5 V), gegenüber ~20 mA bei Direktansteuerung
mit 100 Ω. Basiswiderstand 1,2 kΩ statt 1 kΩ — funktioniert gleichwertig,
der Basisstrom liegt mit ~2,2 mA immer noch weit über dem nötigen Minimum. Der Unterschied ist kein Detail: bei 100 mA reflektiert die LED
von Decke oder Wand stark genug, um zu wirken — sie muss also nicht mehr auf
den Empfänger im Receiver zielen und kann versteckt sitzen. Bei 20 mA
funktioniert nur direkter Sichtkontakt.

Kein Invertierungsproblem wie beim RI-Bus: der Transistor schaltet hier den
eigenen Stromkreis der LED, keine gemeinsam genutzte Signalleitung.

Transistor und Widerstand sind seit 09.09.2026 verbaut, die
Übergangslösung mit 100 Ω direkt an der LED entfällt damit.

Träger 38 kHz, 33 % Tastverhältnis, NEC-Protokoll — Details und alle
verifizierten Codes in `IR-CODES.md`.

## Koax-Ausgang / S/PDIF (GPIO12)

Vollständiges Schema: [spdif-output-schematic.svg](spdif-output-schematic.svg)

```
ESP32-S3 GPIO12 ──[R1 270 Ω]── Knoten A ──[C1 100 nF]── Cinch Mittelpin (Signal)
                                  │
                              [R2 120 Ω]
                                  │
ESP32-S3 GND ─────────────── GND-Schiene ─────────────── Cinch Außenring (Schirm)
```

Mittelpin und Schirm sind zwei getrennte Kontakte der Cinch-Buchse — der
Mittelpin hängt nur über C1 am Signalpfad, der Schirm liegt direkt auf der
GND-Schiene. Nicht verwechseln mit der älteren ASCII-Skizze in früheren
Commits, die beide Leitungen optisch übereinander zeichnete und wie eine
Brücke zwischen Mittelpin und Schirm aussah — das ist elektrisch falsch
und war nur ein Darstellungsfehler, keine reale Verbindung.

Ersetzt den ursprünglich geplanten PCM5102A-DAC vollständig — der R-1045
nimmt am CD/COAX-Eingang koaxiales S/PDIF direkt an, der Receiver wandelt
selbst. BCK und WS werden dabei auf keinen Pin geroutet (Details in
`airplay-esp32/main/audio/audio_output_spdif.c`).

**Wichtig, per 18.09.2026 gefunden:** Der Pin wird über `CONFIG_SPDIF_DO_IO`
konfiguriert, nicht über `CONFIG_I2S_DO_IO` (das ist eine eigene Option, nur
für I2S-DAC-Boards). `CONFIG_SPDIF_DO_IO` steht per Default auf `-1`
("S/PDIF deaktiviert") und muss in `sdkconfig.user.onkyo` /
`sdkconfig.user.onkyo-spdif-only` explizit auf `12` gesetzt werden — sonst
läuft die Firmware fehlerfrei durch (`CONFIG_AUDIO_OUTPUT_SPDIF=y`,
`SPDIF output ready` im Log), aber es liegt kein Signal am Pin an. Dieser
Fehler stand fälschlich in früheren Versionen dieser Doku.

270/120 Ω bilden zusammen mit den 75 Ω des Kabels einen Spannungsteiler, der
aus dem 3,3-V-Logikpegel des ESP32 die S/PDIF-Norm (0,5 V ±20 % an 75 Ω)
macht — rechnerisch rund 0,48 V, mittig im Toleranzband. Der
100-nF-Kondensator entkoppelt Gleichspannung.

Verbaut (09.09.2026): 270 Ω / 120 Ω / 100 nF. Der im Firmware-Quellcode
genannte Ausgangswert 210 Ω/110 Ω ergäbe rund 0,58 V — ebenfalls im Band,
aber näher an der oberen Grenze.

## OLED (GPIO6 / GPIO7)

SSD1306, 128×32, I²C, Adresse `0x3C`. Versorgung **3V3**, nicht 5 V.

```
VCC ── 3V3          SDA ── GPIO6
GND ── GND           SCL ── GPIO7
```

Bestätigt am 30.08.2026 mit dem Bench-Sketch (`pins`-Diagnose: interner
Pulldown als Gegenprobe, genau zwei Pins hochgezogen; danach `i2c auto`
fand `0x3C` auf diesem Pinpaar).

## Bekannter Software-Bug: `/api/remote/send` meldet falsch `false`

Die REST-API (`main/control/remote_control.c`, `main/network/web_server.c`)
antwortet bei praktisch jedem Aufruf mit `"success": false` und
`"error": "ESP_ERR_TIMEOUT"`, meldet also einen Fehler — **der Befehl kommt
aber trotzdem korrekt am Onkyo an.**

Bestätigt am 02.10.2026 durch gezielte Tests mit eindeutig sichtbaren
Befehlen am echten Gerät:

| Befehl | API-Antwort | Tatsächliche Wirkung |
|---|---|---|
| `dimmer` | `ESP_ERR_TIMEOUT` | Helligkeit korrekt geändert |
| `volup` | `ESP_ERR_TIMEOUT` | Lautstärke korrekt erhöht |
| `ri-on` | `ESP_ERR_TIMEOUT` | Gerät korrekt aufgewacht, CD/COAX gewählt |
| `ri-off` | `ESP_ERR_TIMEOUT` | Gerät korrekt in Standby |

Jeder Befehl kam dabei genau einmal und unverfälscht an — keine
Doppel-Sendung, kein falscher Befehl.

**Ursache:** `rmt_transmit()` liefert die Hardware-Seite korrekt aus, aber
`rmt_tx_wait_all_done()` (ESP-IDF-RMT-Treiber) erkennt den Abschluss
fälschlich nicht und meldet `ESP_ERR_TIMEOUT`, obwohl die Übertragung schon
fertig und korrekt war. Mit einem eigenständigen Minimalprojekt (reines
ESP-IDF, nur WLAN + ein HTTP-Endpunkt + ein RMT-Kanal, auch mit einem
zweiten RMT-Kanal für eine Status-LED) ließ sich der Fehler **nicht**
reproduzieren — 32 von 32 Testaufrufen liefen fehlerfrei. Er tritt nur in
der vollen AirPlay-Firmware auf, auch im Leerlauf ohne aktive Wiedergabe.
Verdacht (nicht abschließend verifiziert): der dauerhaft laufende,
kern-gepinnte Audio-Playback-Task (`audio_output_spdif.c` /
`audio_output.c`), der per `i2s_channel_write()` ununterbrochen Daten oder
Stille sendet, stört die RMT-Interrupt-Behandlung.

Ein Retry-Versuch nach dem ersten (fälschlich gemeldeten) Timeout wurde
getestet und wieder entfernt: er hilft nicht (der erste Versuch hat die
Arbeit schon erledigt) und der zweite Versuch scheitert danach selbst
zuverlässig echt.

**Praktische Konsequenz:** Die Fernsteuerung funktioniert schon jetzt
uneingeschränkt für den echten Einsatz. In Home-Assistant-Automationen
sollte das `success`-Feld der API-Antwort nicht als Fehlerindikator
verwendet werden — insbesondere keine Retry-Logik darauf aufbauen, die
sonst unnötig doppelt sendet.

## Power-Status über die RI-Leitung — untersucht, nicht lesbar

Ziel war, den echten An/Standby-Zustand des R-1045 ohne Smart-Plug zu lesen.
Ausgangspunkt war eine Multimeter-Messung am RI-Tip (Pin im Ruhezustand von RMT
aktiv auf Masse, 470 Ω in Reihe): im Betrieb einige mV, im Standby 0 V.

Die Firmware-Variante (`CONFIG_REMOTE_POWER_SENSE`, Code liegt noch in
`remote_control.c`, per Kconfig **aus**) gibt den RI-Pin kurz frei und liest ihn
mit dem ADC (`GET /api/remote/probe` zeigt die Rohdaten). Gemessen am
02.10.2026, Kabel eingesteckt, Zustand jeweils per `ri-on`/`ri-off` gesetzt und
am Gerät bestätigt:

| Pin beim Messen | ADC | Standby | Betrieb |
|---|---|---|---|
| schwebend / interner Pulldown | 0 dB | 1,2–1,3 mV | 0,5–0,8 mV |
| schwebend / interner Pulldown | 12 dB | 0 mV | 0 mV |
| interner Pullup (~45 kΩ) | 12 dB | ≈ 1184,4 mV | ≈ 1182,2 mV |

- Frei schwebend liegt die Leitung bei 0–1 mV — unter dem ADC-Rauschen. (Ohne
  eingestecktes Kabel sieht man dagegen ~50 Hz Netzbrummen mit einigen
  100 mV, aber keinen Pegel.)
- Mit Pullup gibt es einen reproduzierbaren Versatz von ~2,2 mV (4 Aus/Ein-
  Zyklen, Gruppen überlappen nicht). Der Absolutwert driftet aber von Session
  zu Session um mehrere mV (Standby später bei ~1180 mV statt 1184), also mehr
  als der Unterschied. Ein fester Schwellwert funktioniert deshalb nicht.
- Der ESP32-S3-ADC ist bei 12 dB unterhalb von ~100 mV praktisch blind, und
  die Differenz liegt bei 0,2 % des Messwerts.

**Fallstricke, die dabei auftauchten (gelten für jede künftige Variante):**

- `adc_oneshot_config_channel()` ruft intern `gpio_config_as_analog()` auf und
  schaltet damit den **Digitalausgang des Pins ab** — der RMT-Sender, der
  denselben Pin nutzt, sendet danach nichts mehr. Der Pin-Zustand muss vorher
  gesichert und danach zurückgeschrieben werden (`GPIO.func_out_sel_cfg[]`,
  Enable-Bit, `esp_rom_gpio_pad_select_gpio`).
- `gpio_set_direction(pin, OUTPUT)` setzt das GPIO-Matrix-Routing auf das
  einfache GPIO-Signal zurück und trennt damit RMT vom Pin.
- Ohne eingestecktes RI-Kabel ist jede Messung wertlos — und `ri-off`
  scheitert dann scheinbar „lautlos" (die API meldet ohnehin `ESP_ERR_TIMEOUT`).

**Zweiter Versuch mit eigenem ADC-Pin (GPIO2, per Draht am Tip, Jack-Seite des
470-Ω-Widerstands):** GPIO4 treibt dabei unverändert aktiv auf Masse, der ADC
liest nur mit (0 dB, 40 ms gemittelt, drei Aus/Ein-Zyklen). Ergebnis: in beiden
Zuständen ≈ 0,05–0,07 mV (0,2–0,3 ADC-Zählschritte), kein Unterschied.

**Ursache der Multimeter-Werte:** Das Multimeter zeigt die paar mV nur zwischen
**Tip und Sleeve**; Tip gegen den GND-Pin des ESP liest in beiden Zuständen ~0
(0,07 vs 0,08 mV). Der Versatz liegt also zwischen dem Sleeve am Kabel und der
ESP-Masse - ein Ausgleichsstrom zwischen zwei Massen, kein Signal der RI-Leitung.
Er hängt an Netzteil und Erdung des ESP und nur zufällig am Receiver-Zustand.
Eine Differenzmessung Tip-Sleeve an der eigenen Buchse (z. B. ADS1115) würde
ebenfalls 0 lesen, weil dort Sleeve und ESP-Masse derselbe Punkt sind.

**Fazit:** Der An/Standby-Zustand des R-1045 ist über die RI-Leitung vom ESP aus
nicht zuverlässig lesbar. Bleibt: Smart-Plug mit Leistungsmessung, oder dem
zuletzt gesendeten RI-Befehl trauen. `CONFIG_REMOTE_POWER_SENSE` bleibt als
dokumentiertes Experiment im Code, in `sdkconfig.user.onkyo` aus.

## Stückliste

| Bauteil | Wert | Zweck | Status |
|---|---|---|---|
| Widerstand | 470 Ω | RI-Schutz | ✓ verbaut |
| Widerstand | 270 Ω | S/PDIF-Anpassung (R1) | ✓ verbaut |
| Widerstand | 120 Ω | S/PDIF-Anpassung (R2) | ✓ verbaut |
| Kondensator | 100 nF | S/PDIF-Entkopplung (C1) | ✓ verbaut |
| Widerstand | 33 Ω | IR-LED-Vorwiderstand | ✓ verbaut |
| Widerstand | 1,2 kΩ | IR-Basiswiderstand | ✓ verbaut |
| NPN-Transistor | BC337 oder 2N2222 | IR-Treiber | ✓ verbaut |
| IR-LED | 940 nm, TSHG6400 | Sender | ✓ verbaut (Tausch auf TSAL6100 vorgeschlagen, Status Tausch unbestätigt) |
| Einbaubuchse | Cinch | S/PDIF-Ausgang | ✓ verbaut |
| Einbaubuchse | 3,5 mm mono | RI-Ausgang | ✓ verbaut |
| OLED-Modul | SSD1306 128×32 I²C | Anzeige | ✓ vorhanden, verifiziert |

Entfallen mit dem Umstieg auf S/PDIF: PCM5102A-DAC, dessen I2S-Verkabelung
(GPIO11/12/13 alt, 5V, GND), VIN/VOUT-Lötbrücke.
