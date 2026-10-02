# Onkyo R-1045 — IR-Codebuch

Aufgenommen von der Originalfernbedienung **RC-789S** mit
`esphome/onkyo-ir-sniffer.yaml` (`remote_receiver` auf GPIO5, `dump: all`).

Protokoll ist **NEC** mit erweiterter 16-Bit-Adresse. Alle Prüfsummen der
32-Bit-Rahmen sind gültig.

Die Werte sind so notiert, wie **ESPHomes eigener NEC-Dekoder** sie ausgibt.
Genau so gehören sie auch in `remote_transmitter.transmit_nec` — dieselbe
Komponente, dieselbe Konvention, kein Umrechnen. Wer die Rohrahmen braucht:
in der Spalte rechts steht der 32-Bit-Rahmen MSB-first.

## Bestätigte Codes

| Taste | Adresse | Kommando | Rahmen (MSB first) |
|---|---|---|---|
| **OPT** | `0x03D2` | `0xA857` | `0x4BC0EA15` |
| **VOLUME ▲** | `0x03D2` | `0xFD02` | `0x4BC040BF` |
| **VOLUME ▼** | `0x03D2` | `0xFC03` | `0x4BC0C03F` |
| **MUTING** | `0x03D2` | `0xFA05` | `0x4BC0A05F` |
| LINE 1 | `0x03D2` | `0xF906` | `0x4BC0609F` |
| LINE 2 | `0x03D2` | `0xEE11` | `0x4BC08877` |
| LINE 3 | `0x04D2` | `0xB44B` | `0x4B20D22D` |
| CD/COAX | `0x03D2` | `0xA956` | `0x4BC06A95` |
| TUNER | `0x03D2` | `0xF40B` | `0x4BC0D02F` |
| PHONO | `0x03D2` | `0xF50A` | `0x4BC050AF` |
| iPod / USB | `0x05D2` | `0x619E` | `0x4BA07986` |
| ON/STANDBY | `0x04D2` | `0x34CB` | `0x4B20D32C` |
| DIMMER | `0x03D2` | `0x6A95` | `0x4BC0A956` |

Fett markiert sind die vier, die der Aufbau zwingend braucht.

**Alle Einträge sind am Gerät verifiziert (30.08.2026):** jeder Code wurde
über die selbstgebaute IR-Strecke gesendet und die erwartete Reaktion am
R-1045 kontrolliert. Damit ist nicht nur der Code belegt, sondern auch seine
Beschriftung — die beruhte vorher auf der Reihenfolge beim Aufnehmen.

## Belegung der Eingänge

| Eingang | Verwendung | Adresse | Kommando |
|---|---|---|---|
| **OPT** | Fernseher, optisch | `0x03D2` | `0xA857` |
| CD/COAX | **AirPlay, koaxial digital** | `0x03D2` | `0xA956` |
| LINE 1 | Plattenspieler | `0x03D2` | `0xF906` |
| LINE 2 | frei | `0x03D2` | `0xEE11` |
| LINE 3 | frei | `0x04D2` | `0xB44B` |

**AirPlay liegt auf CD/COAX, und das hat einen Grund, der über die
Bequemlichkeit hinausgeht:** Der RI-Befehl `0x20` wählt genau diesen Eingang
*und* weckt das Gerät dabei. Für den kompletten AirPlay-Vorgang genügt damit
ein einziger Befehl über das Kabel — kein IR-Kommando, keine Wartezeit
dazwischen, keine Sichtlinie.

Der Eingang nimmt außerdem koaxiales S/PDIF entgegen. Der ESP32 gibt das
direkt aus (`AUDIO_OUTPUT_SPDIF`), der Receiver wandelt selbst — ein
D/A-Wandler im Gehäuse ist damit überflüssig.

Für den TV-Modus bleibt `OPT` über IR. Beide Umschaltungen sind idempotent:
läuft der Receiver schon auf dem Eingang, passiert nichts.

## DIMMER und CD/COAX sehen vertauscht aus, sind es aber nicht

```
CD/COAX   LG 0x4BC06A95   →  NEC cmd 0xA956
DIMMER    LG 0x4BC0A956   →  NEC cmd 0x6A95
```

Die Hexziffern tauchen über Kreuz auf, weil NEC bitweise umgekehrt gelesen
wird und die beiden Kommandobytes zueinander bit-verkehrt sind. Beide Rahmen
haben gültige Prüfsummen, es sind zwei verschiedene Codes. Wer die Tabelle
später überfliegt, stolpert hier sonst über eine vermeintliche Verwechslung.

DIMMER schaltet die Displayhelligkeit stufenweise weiter — jeder Druck sendet
denselben Code, das Gerät zählt selbst durch.

## ON/STANDBY ist ein Toggle

Zweimal gedrückt, zweimal derselbe Code (`0x04D2` / `0x34CB`, Zeitstempel
01:56:47.763 und 01:56:48.565, also 800 ms auseinander und damit sicher zwei
getrennte Drücke, kein Wiederholrahmen).

**Damit bleibt die Architekturentscheidung bestehen:** Ein- und Ausschalten
läuft über RI (`0x20` / `0x420`, beide diskret), nicht über IR. Ein Toggle
ohne Rückkanal führt früher oder später dazu, dass der „Aus"-Befehl einschaltet.

## Drei verschiedene Geräteadressen

Die meisten Tasten senden auf `0x03D2`, aber drei nicht:

| Adresse | Tasten |
|---|---|
| `0x03D2` | OPT, Volume, Mute, LINE 1, LINE 2, CD/COAX, TUNER, PHONO |
| `0x04D2` | LINE 3, ON/STANDBY |
| `0x05D2` | iPod / USB |

Onkyo-Fernbedienungen adressieren verschiedene Gerätekategorien getrennt, das
ist an sich normal. Auffällig war, dass **LINE 3 und ON/STANDBY sich eine
Adresse teilen und ihre Kommandos benachbart sind** — im Rohrahmen `0xD2` und
`0xD3`. Genau die Stelle, an der eine verrutschte Zuordnung unbemerkt bliebe.

**Gegengeprüft und bestätigt:** In einer zweiten Aufnahme wurden LINE 1, LINE 2,
LINE 3 und ON/STANDBY je dreimal einzeln gedrückt, mit mehreren Sekunden Pause.
Jeder Block lieferte dreimal denselben Code, deckungsgleich mit der ersten
Aufnahme. Die geteilte Adresse ist also echt und kein Aufnahmefehler.

## Verlässlichkeit der Zuordnung

Die Codes selbst sind sicher: saubere Rahmen, gültige Prüfsummen. Die
**Beschriftung** beruht darauf, dass beim Aufnehmen die vorgegebene Reihenfolge
eingehalten wurde. Dafür spricht:

- 13 unterscheidbare Tastendrücke für 13 Listeneinträge
- Anfang (OPT) und Ende (ON/STANDBY zweimal) sind eindeutig verankert
- Die Zeitabstände passen zur Tastenanordnung: LINE 1/2/3 kamen im Abstand von
  je ~640 ms, also zügig entlang einer Reihe; danach eine Pause vor CD/COAX

**Erledigt.** Alle dreizehn Codes wurden gesendet und die Reaktion am Gerät
kontrolliert. Die Zuordnung aus der Aufnahmereihenfolge hat über die ganze
Liste gehalten, auch bei den Einträgen mit abweichender Geräteadresse.

Die Tabelle ist damit belastbar und kann so in die Firmware übernommen werden.
