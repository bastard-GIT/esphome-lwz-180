# esphome-lwz-180

[English](README.md) | Deutsch

ESPHome-Paket für das Lüftungsgerät **Stiebel Eltron LWZ 180**. Der ESP hängt als I2C-Slave am Bus des Bedienteils, liest Istwerte und Display mit und meldet sie per MQTT-Discovery als eigenes Home-Assistant-Gerät. Über ein HA-Select und einen HA-Switch kann er die Lüfterstufe setzen und die Intensivlüftung starten.

Die Analyse des I2C-Busses stammt aus dem Vorprojekt [kdarnok/lwz180](https://github.com/kdarnok/lwz180/), siehe Abschnitt Dank.

Das Paket läuft auf jedem ESP32 mit esp-idf und MQTT. Getestet: ESP32-S3, ESP-IDF 5.5, ESPHome 2026.10.

## Hardware

- ESP als **Hardware-I2C-Slave** am Bedienteil-Bus zwischen Bedienteil und LWZ-Steuerung (Slave-Adresse 42, dazu General Call, kein Clock-Stretching).
- Galvanische Trennung über eine Trägerplatine mit **ISO1540**. ESP-Seite: SDA GPIO15, SCL GPIO17 (Beispiel, per Substitution änderbar). Die ESP-seitigen Pull-ups sind nicht bestückt, die internen Pull-ups des ESP32-S3 reichen. Auf der LWZ-Seite wirken die Pull-ups der LWZ.
- Der ESP schreibt nur kurz als Master, wenn er Tasten sendet (Lüfterstufe, Intensivlüftung). Dazu baut er den Slave ab und danach wieder auf.

## Einbinden

Remote, in der eigenen ESPHome-YAML:

```yaml
packages:
  lwz_180: github://bastard-GIT/esphome-lwz-180/lwz_180.yaml@main
```

Für stabile Builds statt `@main` einen Tag oder Commit-Hash verwenden. Ein vollständiges Beispiel steht in `example/lwz_180_example.yaml` (lokal eingebunden mit `!include ../lwz_180.yaml`). Dort liegt auch `secrets.yaml.example`.

## Voraussetzungen

- ESP32 mit `esp32: framework: type: esp-idf`.
- `mqtt:` Werte, Discovery und Befehle laufen über MQTT, nicht über die Native API.
- `ota:` mit `platform: esphome` und `id: ota_esphome` (oder eine andere id, dann `lwz_ota_id` setzen). Das Paket erweitert diese Komponente per `!extend`. Der Slave wird während OTA abgebaut.

Das Paket setzt selbst die nötigen `sdkconfig_options` (I2C-Slave-Treiber V2, `CONFIG_I2C_ISR_IRAM_SAFE`, Ringbuffer-Funktionen im IRAM) und bindet seine Header aus dem Ordner `lwz_180/` ein.

## Substitutionen

Standardwerte, in der eigenen YAML unter `substitutions:` überschreibbar.

| Substitution | Standard | Bedeutung |
|---|---|---|
| `lwz_sda_gpio` | `15` | SDA-GPIO (Nummer ohne "GPIO") |
| `lwz_scl_gpio` | `17` | SCL-GPIO |
| `lwz_i2c_slave_addr` | `42` | 7-Bit-Adresse des ESP als Slave |
| `lwz_internal_pullup` | `true` | interne Pull-ups |
| `lwz_prefix` | `lwz180` | MQTT- und ID-Präfix der LWZ-Werte |
| `lwz_device_name` | `Stiebel Eltron LWZ 180` | HA-Gerätename |
| `lwz_model` | `LWZ 180` | HA-Modell |
| `lwz_availability_topic` | `""` | leer = Availability-Topic der MQTT-Komponente (Standard `<topic_prefix>/status`), sonst eigenes Topic, z. B. `heatingpump/status` |
| `lwz_ota_id` | `ota_esphome` | id der OTA-Komponente |

## MQTT-Topics

| Topic | Inhalt |
|---|---|
| `<prefix>/<wert>/state` | Istwerte |
| `<prefix>/availability` | online, solange gültige LWZ-Pakete kommen (offline nach 15 min ohne Paket) |
| `<prefix>/luefterstufe/set` | Lüfterstufe setzen: `0` bis `2`, `3` = Intensivlüftung an |
| `<prefix>/intensivlueftung/set` | `ON` / `OFF` |
| `<prefix>/intensivlueftung/state` | Zustand (retained) |
| `<prefix>/steuerung/result` | Ergebnistext des letzten Auftrags (nicht retained) |
| `homeassistant/sensor/<prefix>/<wert>/config` | MQTT-Discovery |

## Home-Assistant-Entitäten

Alle gehören zu einem Gerät. Sensoren (Entitäten entstehen beim ersten Empfang des Werts):

- Temperaturen: Außenlufttemperatur, Zulufttemperatur, Ablufttemperatur, Fortlufttemperatur, Taupunkt Abluft, Taupunkt Außenluft
- Feuchte Außenluft, Feuchte Abluft
- Volumenstrom Zuluft, Volumenstrom Fortluft (m³/h)
- Zuluftlüfter Ansteuerung, Fortluftlüfter Ansteuerung (%), Zuluftlüfter Drehzahl, Fortluftlüfter Drehzahl (rpm)
- Lüfterstufe (aus dem Display), Lüfterstufe Einstellung (aus Typ 02; kommt nur bei einem Stufenwechsel, 0,4-1,5 s nach dem Display)
- Volumenstrom Soll Zuluft, Volumenstrom Soll Abluft (m³/h; auf unserem Bus bestätigt)
- Fühlerwiderstand Fortluft, Fühlerwiderstand Zuluft (Ω, PT1000; auf unserem Bus bestätigt; PR #2 nennt 35 Abluft, unsere Logs zeigen Fortluft)
- Auf unserem Bus bisher nicht gesehen (aus PR #2 von kdarnok/lwz180): Bypass (0/1), Volumenstrom Stufe 0-3 (P6-P9, m³/h), Druck P10-P13 (Pa)
- Filterlaufzeit (h, stündlich), Betriebsdauer Gerät, Betriebsdauer Lüfter (d; nach kdarnok, auf unserem Bus bestätigt 2026-11-11)
- Vorheizung Leistung (nur nach Fremdquelle bekannt, auf unserem Bus bisher nicht gesehen)
- Binär-Sensoren (alle nur im Grundzustand ausgewertet, erst nach 60 s ohne Symbol aus (Symbole blinken vermutlich), per MQTT-Discovery erst nach dem ersten Grundzustand-Paket): Filterwechsel (Filtersymbol, bestätigt, device_class problem), Bypass aktiv (bestätigt), Zeitprogramm (Uhr-Symbol, bestätigt), Intensivlüftung (Sketch/Power Vent, nicht unabhängig bestätigt), Frostschutz (unbestätigt), Tastensperre (Schloss, unbestätigt)

Steuerung:

- Select **Lüfterstufe einstellen** (Optionen 0 bis 3)
- Switch **Intensivlüftung**

## Bedienung der Lüfterstufe

Der ESP drückt Tasten wie das Bedienteil und prüft das Ergebnis im Display (geschlossener Regelkreis).

- Stufe 0 bis 2: Drehrad-Burst, 5 Pakete im Abstand von 100 ms (Code `02` = auf, `01` = ab) ergeben eine Stufe. Etwa jeder vierte Burst wirkt nicht, meist nach einem Richtungswechsel. Deshalb prüft die Firmware die Stufe im Display-Paket und wiederholt den Burst, höchstens 8-mal, höchstens 60 s je Auftrag.
- Stufe 3 ist per Drehrad nicht erreichbar, nur als Intensivlüftung. Das Select-Element "3" startet sie.
- Intensivlüftung ein: Power-Vent lang (`08`), nach 500 ms `0c`. Sie läuft bis zum Ende des Timers. Vorzeitig beenden geht nur durch Verstellen der Stufe, der Switch AUS sendet dazu einen Burst ab.
- Gesendet wird nur, wenn das Display den Grundzustand zeigt, also kein Menü offen ist. Sonst würde das Drehrad im Menü blättern.

## Diagnose

Topics `stiebel-i2c/*`: `status` (alle 5 s), `rx` (Sniffer, Standard aus), `crash`, Befehle `cmd/sniff`, `cmd/scan`, `cmd/key`. `cmd/key` sendet Tastencodes auf den Bus, mit Vorsicht benutzen.

## Dank

Grundlage für dieses Paket ist das Projekt [kdarnok/lwz180](https://github.com/kdarnok/lwz180/). Es hat den I2C-Bus der LWZ 180 analysiert und dokumentiert: Paketformat mit CRC, Istwerte, Tastencodes des Bedienteils und die Kodierung des Displays. Vielen Dank an kdarnok für diese Vorarbeit.

Ebenfalls Dank an KKSzymanowski für die erweiterte Tabelle der Istwert-Typen ([PR #2](https://github.com/kdarnok/lwz180/pull/2)): Lüfterstufe, Soll-Volumenstrom, Fühlerwiderstand, Bypass, Drücke.

Dieses Paket enthält keinen Code aus kdarnok/lwz180. Es ist eine eigene Umsetzung für ESP32 und ESPHome und nutzt die dort dokumentierten Erkenntnisse.

## KI-Unterstützung

Code, Dokumentation und Auswerte-Werkzeuge dieses Pakets wurden mit Unterstützung von KI erstellt (Claude von Anthropic, über Claude Code). Die Protokoll-Analyse beruht auf eigenen Messungen am Gerät (Logic-Analyzer- und Bus-Logs) und den unter Dank genannten Quellen. Die Firmware läuft beim Autor auf echter Hardware (ESP32-S3, Stiebel Eltron LWZ 180) und ist dort getestet. Einsatz auf eigene Verantwortung. Bei Tasten- und Schreibbefehlen auf den Bus ist Vorsicht geboten.

## Lizenz

GPL-3.0, siehe [LICENSE](LICENSE). Die I2C-Busanalyse, auf der dieses Paket aufbaut, stammt von kdarnok/lwz180 (GPL-2.0); es wurde kein Code daraus übernommen.
