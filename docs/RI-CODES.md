# Onkyo R-1045 — RI-Codebuch

Arbeitsstand der Code-Suche am echten Gerät. Getestet mit
`tools/onkyo_ri_tester/onkyo_ri_tester.ino` über den RI-Bus
(GPIO10 → 470 Ω → Tip, GND → Sleeve).

Alle Codes sind 12 Bit, MSB first. Frame: Header 3000/1000 µs,
Bit 1 = 1000/2000, Bit 0 = 1000/1000, Footer 1000 + 20 ms Pause.

---

## Wichtig: Standby verfälscht Messungen

Der R-1045 ignoriert im Standby fast alles außer Eingangsbefehlen. Ein Code,
der dort nichts tut, kann am eingeschalteten Gerät sehr wohl wirken.

`0x421` stand hier als widerlegt — im Standby getestet. Am laufenden Gerät tut
es etwas. **Jedes Negativergebnis aus dem Standby ist damit wertlos**, und das
betrifft den größten Teil der ursprünglichen Widerlegt-Liste.

Betroffen waren die gezielten Einschaltcode-Tests — dort lag das Gerät
zwangsläufig im Standby. **Nicht** betroffen sind die Sweeps: der Receiver ist
während keines Durchlaufs von allein abgeschaltet.

Regel ab sofort: **alles am eingeschalteten Gerät testen**, und beim Sweep
`reset 20` setzen, damit Eingangswechsel sichtbar bleiben.

## Bestätigt am R-1045

| Code | Funktion | Belegt durch |
|---|---|---|
| `0x20` | **Eingang CD/COAX** — weckt dabei aus dem Standby | Display zeigt „CD/COAX" |
| `0x420` | **Standby** | direkt am Gerät verifiziert |
| `0x421` | Testton Kanal 1 — Display „test-1-00" | Sweep am eingeschalteten Gerät |
| `0x422` | Testton Kanal 2 — Display „test-2-00" | Sweep am eingeschalteten Gerät |
| `0x423` | Testton Kanal 3 — Display „test-3-00" | Sweep am eingeschalteten Gerät |
| `0x424` | Display „Clear" — **nicht wiederholen** | Sweep am eingeschalteten Gerät |
| `0x425` | schaltet aus | Sweep am eingeschalteten Gerät |

### Achtung: `0x420`–`0x425` ist Service-/Setup-Ebene

Testtongenerator mit Pegelanzeige (`test-N-00`, teils mit leisem Ton) und
direkt daneben ein „Clear". Das sind Einrichtungswerkzeuge, keine
Bedienbefehle.

- **`0x424` nicht erneut senden.** Bestätigt durch die Fremddokumentation: beim
  TX-8020 heißt es zu `0x421`–`0x424` ausdrücklich *„Test modes provides clear
  of receiver setting."* Beim TX-SR333 zeigen dieselben Codes *„Test 1-00,
  2-00, 3-00, 4-00"* — wortgleich mit dem, was am R-1045 zu sehen war.
- **Kanalpegel gegenhören.** Falls `0x421`–`0x423` Pegel gesetzt statt nur
  angezeigt haben, kann die Balance verstellt sein.
- Beim weiteren Absuchen diesen Block überspringen: `sweep 426 4FF`.

`0x20` ist damit geklärt: **kein Power-Befehl, sondern Eingangswahl.** Das
Einschalten ist der Nebeneffekt — Onkyo-Geräte wachen auf, wenn sie einen
Eingangsbefehl bekommen.

Das ist die wichtigste Erkenntnis bisher, denn es heißt: **jeder gefundene
Eingangscode ist gleichzeitig ein Einschaltbefehl.** Findet sich der Code für
den optischen Eingang, braucht das Power-Sync-Blueprint keine Sequenz aus
Einschalten, Warten und Umschalten mehr, sondern genau einen Befehl.

## Widerlegt

| Code | Erwartung | Ergebnis |
|---|---|---|
| `0x1A2` | **Volume Up** | keine Reaktion, Gerät eingeschaltet |
| `0x1A3` | **Volume Down** | keine Reaktion, Gerät eingeschaltet |
| `0x120` | Eingang optisch (BD/DVD, andere Modelle) | keine Reaktion |
| `0x1A2` | **Volume Up** | keine Reaktion |
| `0x1A3` | **Volume Down** | keine Reaktion |
| `0x2F` | Power On (andere Modelle) | keine Reaktion |
| `0x4` | Power Toggle (A-803) | keine Reaktion |
| `0x1AF` | Power On (TX-SR) | keine Reaktion |
| `0x41F` | Power On, Nachbarschaft `0x420` | keine Reaktion |
| `0x42F` | dito | keine Reaktion |
| `0x70` | Eingang Tape / Aux | keine Reaktion |
| `0x120` | Eingang optisch (BD/DVD) | keine Reaktion |
| `0x170` | Eingang Dock / iPod | keine Reaktion |
| `0x0E9` | `AMP_ON` (LIRC) | keine Reaktion |
| `0x0EA` | `AMP_Standby` (LIRC) | keine Reaktion |
| `0xAA0` | `AMP_2` — Volume Down? (LIRC) | keine Reaktion |
| `0xAA1` | `AMP_Muting` (LIRC) | keine Reaktion |
| `0xAA2` | `AMP_1` — Volume Up? (LIRC) | keine Reaktion |
| `0x1B2` | `AMP_Dimmer` (LIRC) | keine Reaktion |

**Alle am eingeschalteten Gerät geprüft.** Die acht Codes, die ursprünglich im
Standby getestet und dadurch wertlos widerlegt worden waren, wurden nachgeholt
— sie tun auch unter richtigen Bedingungen nichts. Die übernommenen
Community-Listen sind damit endgültig erledigt.

### Konsequenz: die übernommene Liste ist unbrauchbar

Elf geratene Codes, kein einziger Treffer. `0x1A2`, `0x1A3` und `0x1AF` kommen
aus TX-SR-Listen, `0x70`/`0x120`/`0x170` aus anderen Eingangs-Tabellen, `0x2F`
aus einer dritten Quelle. Am R-1045 tut keiner davon irgendetwas.

**Raten ist damit beendet.** Ab hier wird nur noch systematisch gesucht.

### Systematisch abgesucht, ohne Treffer

| Bereich | Codes | Ergebnis |
|---|---|---|
| `0x000`–`0x0FF` | 256 | nur `0x20`, sonst nichts |
| `0x_20` (Muster-Sweep) | 16 | nichts Neues |

Der Muster-Sweep prüfte die Hypothese, das untere Byte sei eine Geräteadresse
und die oberen Bits die Aktion — `0x020` und `0x420` legten das nahe. Sie ist
**widerlegt**: von `0x020` bis `0xF20` tut nur das bereits bekannte `0x020`
etwas.

Die Sendestrecke ist dabei nachgewiesen: Eingang von Hand gewechselt, `0x20`
gesendet, Display springt zurück auf CD/COAX. Leere Sweeps heißen also
tatsächlich „diese Codes bewirken nichts" und nicht „Kabel lose".

### Verbleibender Suchraum

`0x100`–`0xFFF` abzüglich der einzeln geprüften Codes — rund 3800 Stück.

Damit steht auch die Lautstärkesteuerung ohne Code da, und das ist der
eigentliche Zweck der ganzen Kette. Sie hat ab jetzt Priorität vor dem
optischen Eingang.

---

## Abdeckung des Coderaums

| Bereich | Codes | Wie geprüft | Ergebnis |
|---|---|---|---|
| `0x000`–`0x0FF` | 256 | Sweep, ohne Reset | nur `0x20` |
| `0x100`–`0x455` | 854 | Sweep 120 ms, ohne Reset | nur der `0x42x`-Block |
| `0x400`–`0x4FF` | 256 | Sweep mit `reset 20` | `0x420`–`0x425`, sonst nichts |
| `0x_20` | 16 | Muster-Sweep | nichts Neues |
| `0x500`–`0xFFF` | 2816 | Sweep 120 ms, ohne Reset | nichts |

**Der gesamte 12-Bit-Raum ist abgesucht.**

## Ergebnis: RI trägt an diesem Gerät keine Lautstärke

Von 4096 möglichen Codes reagieren sieben — Eingangswahl, Ein/Aus und ein
Service-Block. Keine Lautstärke, kein Mute, keine weiteren Eingänge.

Das Ergebnis ist belastbar: derselbe schnelle Sweep, der zuletzt leer ausging,
hat zuvor den `0x42x`-Block gefunden. Die Methode erkennt Treffer nachweislich,
ein leeres Ergebnis heißt hier also tatsächlich „da ist nichts".

Auch die vollständige Onkyo-RI-Referenztabelle wurde gegengeprüft (`0x070`,
`0x07F`, `0x120`, `0x12F`, `0x1A0`, `0x1A2`–`0x1A5`, `0x1AE`, `0x1AF`) — ohne
Reaktion.

Ebenso die **AMP-Gruppe aus der LIRC-Konfiguration** für Onkyo RI
(`lirc.sourceforge.net/remotes/onkyo/Remote_Interactive`): `AMP_ON`,
`AMP_Standby`, `AMP_Muting`, `AMP_1`, `AMP_2`, `AMP_Dimmer`. Einzeln und
langsam getestet, am eingeschalteten Gerät — nichts. Das waren die am besten
begründeten Kandidaten überhaupt, aus einer echten Protokolldatei statt aus
zusammengetragenen Listen.

Dieselbe Datei bestätigt nebenbei unseren Frame exakt: `bits 12`,
`header 3000 1000`, `one 1000 2000`, `zero 1000 1000`, `ptrail 1000`. Einziger
Unterschied ist die Pause zwischen zwei Frames — LIRC nennt 67 ms, der Tester
verwendet 20 ms. Für Einzelbefehle ohne Belang. Das Protokoll stimmt, der R-1045 implementiert davon nur einen
Bruchteil. Plausibel für einen Mini-Receiver: RI ist dort die
Verknüpfungsleitung zum CD-Player, kein vollwertiger Fernsteuerbus.

### Warum der Coderaum so leer ist

Onkyo beschreibt für die 1045-Serie drei RI-Systemfunktionen zwischen R-1045
und C-1045 CD-Player:

| Beschriebene Funktion | Gefundener Code |
|---|---|
| Auto Power On — CD-Player weckt den Receiver | `0x20` |
| Direct Change — Play schaltet auf CD-Eingang | `0x20` |
| System Off — Receiver schickt Standby weiter | `0x420` |

**Lautstärke kommt in dieser Funktionsliste nicht vor.** RI ist hier kein
Fernsteuerbus, sondern eine Dreifach-Verknüpfung zwischen zwei Geräten. Die
Suche hat also nichts übersehen — sie hat den vollständigen Funktionsumfang
gefunden.

Einschränkung zur Quelle: die auffindbare Onkyo-Dokumentation beschreibt den
A-9110, nicht den R-1045. Sie erklärt unsere Messung, belegt sie aber nicht.
Die Messung selbst ist die stärkere Evidenz.

Nebenbefund: „System Off" heißt, der Receiver **sendet** selbst auf dem Bus.
Ein Mithören am RI-Bus würde also Daten liefern, falls das später gebraucht
wird.

### Offen: Lautstärke abhängig vom aktiven Eingang

**Der Sweep hat einen blinden Fleck.** Er lief durchgehend, während der Receiver
auf CD/COAX stand — erzwungen durch `reset 20`, und weil `0x20` der
Aufweckbefehl war. Getestet wurden also 4096 Codes, aber nur in **einem**
Gerätezustand.

Die Dokumentation von docbender/Onkyo-RI zeigt, dass das ein Problem sein kann.
Bei TX-SR504, TX-SR600, TX-SR603, TX-SR606 und HT-R340 gilt:

> Vol Up `0x1A2` — *only when input is set to Video 3*
> Power Off `0x1AE` — *only works if you are currently in Video 3!!*

Beim HT-R340 steht zusätzlich die ausdrückliche Warnung, dass ein
funktionierender Eingangs-Präfix plus bekannter Suffix **nicht** automatisch
einen gültigen Lautstärkebefehl ergibt.

**Getestet (30.08.2026):** `0x1A2` an allen acht Eingängen (OPT, LINE 1,
LINE 2, LINE 3, CD/COAX, TUNER, PHONO, iPod) — **keine Reaktion.** Die
Eingangsabhängigkeit, die bei anderen Onkyo-Modellen die Lautstärke rettet,
trägt beim R-1045 nicht.

**Bestätigt: die A-9050-Erklärung trifft zu.** Dieselbe Doku, für ein Gerät
derselben kompakten Klasse:

> Die Lautstärkecodes werden vom Receiver über RI *ausgegeben*, wenn man am
> Gerät die Lautstärke ändert. Er reagiert aber nicht darauf, wenn sie von
> außen kommen.

Der R-1045 sendet Lautstärke über RI, nimmt sie aber nicht entgegen. Damit ist
die Suche nach einem RI-Lautstärkecode endgültig abgeschlossen — **die
Architekturentscheidung (Lautstärke über IR) steht fest, nicht mehr nur
vorläufig.**

### Konsequenz für den Aufbau

Hybrid statt Entweder-oder:

- **Ein/Aus und Eingang über RI** — zuverlässig, kein Sichtkontakt nötig
- **Lautstärke über Infrarot** — denselben Weg, den die Originalfernbedienung
  nutzt

Die Bauteile dafür stecken bereits in einem vorhandenen IR-Verlängerungskabel
(Empfängerauge + Sende-LED). Siehe `tools/onkyo_ir_sniffer/`.

## Als Nächstes testen

**1 — Lautstärke finden.** Die wichtigste offene Frage. Das Display zeigt den
Lautstärkewert an, du brauchst also keine Musik — es reicht, auf die Zahl zu
schauen. Gerät einschalten, dann den unteren Bereich absuchen, in dem auch die
beiden funktionierenden Codes liegen:

```
sweep 0 FF
```

256 Codes, gut sechs Minuten. Der Sweep überstreicht dabei `0x20`, springt also
zwischendurch auf CD/COAX — das ist normal. Findet sich nichts, weiter mit
`sweep 100 1FF` und `sweep 400 4FF`.

**2 — Den optischen Eingang finden.** Höchste Priorität: das ist der Code, den
die ganze Kette am Ende braucht. Aus dem Standby heraus testen, nach jedem
Versuch das Display ablesen und mit `0x420` wieder ausschalten.

```
70      Tape / Aux?
170     Dock / iPod?
```

(`120` ist bereits durch — keine Reaktion.)

Greift keiner davon, den Bereich um `0x20` absuchen — die Eingänge liegen
erfahrungsgemäß dicht beieinander:

```
sweep 0 FF
```

**3 — Volume gegenprüfen, Gerät eingeschaltet.** `1A2` und `1A3`. Siehe oben,
das ist keine Formalie.

**3 — Mute.** Erst sinnvoll, wenn die Volume-Familie gefunden ist — Mute liegt
vermutlich direkt daneben.

**4 — Dimmer.** `2B0` hell, `2B1` mittel, `2B2` dunkel, `2B8` Tag, `2BF` Nacht.

**5 — Weitere Eingänge.** `B0` Tuner, `A` Phono, `6` Aux/Video, `8` Tape-1,
`9` CD, `E0` Line 2, `D5` Input Next, `D6` Input Previous.

**6 — Auffangnetz.** Alles, was dann noch fehlt:

```
sweep 0 FFF 150
```

Der Zustand bleibt verändert, wenn etwas greift — du musst nicht mitlesen,
nur hinschauen und bei einer Reaktion Enter drücken.

---

## Kommandos des Testers

| Eingabe | Wirkung |
|---|---|
| `1A2` | Code einmal senden |
| `5x 1A2` | fünfmal senden |
| `sweep 20 30` | Bereich durchgehen, 1,5 s pro Code |
| `sweep 0 FFF 150` | mit eigenem Abstand in ms (min. 60) |
| `ok <Name>` | letzten Code als bestätigt notieren |
| `list` | Notiertes als Markdown-Tabelle + passende `ri_raw.py`-Aufrufe |
| `high` / `low` | Pin dauerhaft treiben, zum Messen |
| `square` | 1 kHz Rechteck für Oszi/Logic Analyzer |
| `led` | Status-LED an/aus |
| `?` | Hilfe |

## Hardware-Fakten

- **Ruhepegel am RI-Bus:** ~1,5 mV. Kein interner Pull-up, der ESP32 muss
  aktiv treiben. Ein NPN in Emitterschaltung funktioniert hier nicht.
- **3,3 V reichen** dem R-1045 als High — durch `0x420` und `0x20` bewiesen.
- **Zwei RI-Buchsen, funktional gekoppelt.** Am Verhalten geprüft: umgesteckt,
  dieselben Codes wirken, dieselben wirken nicht. Es gibt also keine getrennte
  RI-IN/RI-OUT-Leitung, an der noch etwas anderes zu holen wäre — die
  Negativergebnisse gelten fürs ganze Gerät.
- **Board:** ESP32-S3-DevKitC-1, WROOM-1 N16R8. In der Arduino IDE
  „ESP32S3 Dev Module", USB CDC On Boot = Disabled, PSRAM = OPI, Flash 16 MB.
