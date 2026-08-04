# Marstek Venus Modbus Implementation mit ESP32 und ESPHome

Lokale Steuerung und Überwachung eines **Marstek Venus** Speichers mit **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** und **Home Assistant**.

<a href="README.md"><img src="https://flagcdn.com/24x18/gb.png" width="24" height="18" alt="English"> English README</a>

[![ESPHome](https://img.shields.io/badge/ESPHome-ESP32-blue)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-compatible-blue)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Hiermit lässt sich die Ladung/ Entladung eines Martek Venus direkt aus Home Assistant heraus komfortabel steuern. Es ist keine Cloud-Verbindung und nicht die Marstek "Local API" nötig. Der TTGO erhält die Befehle per WLAN von Home Assistant und leitet sie per Modbus weiter. Er stellt in Home Assistant Sensoren, Diagnosewerte sowie Lade- und Entlade-Sollwerte bereit. Das TTGO-Display zeigt die wichtigsten Informationen direkt am Gerät.

> **Projektstatus: abgeschlossen / Referenzimplementierung.** Getestet mit einem Marstek Venus E Gen3 und Firmware V148. Das Projekt wird nicht aktiv weiterentwickelt. Anpassungen erfolgen nur bei Bedarf, beispielsweise wenn Änderungen der Marstek-Firmware die RS485-/Modbus-Kommunikation beeinflussen. Nach Firmwareupdates des Speichers sollte die Steuerung deshalb immer erneut geprüft werden.

## Der fertige Aufbau

<p align="center"><img src="images/wall_setup.jpg" alt="Marstek Venus mit TTGO-RS485-Steuerung" width="900"></p>

Der kleine TTGO-Controller sitzt direkt neben dem Marstek Venus und übernimmt die lokale RS485-Kommunikation.

<p align="center"><img src="images/case.jpg" alt="3D-gedrucktes TTGO Gehäuse" width="900"></p>

## Warum dieses Projekt?

Der eigentliche Grund für dieses Projekt ist nicht nur, Daten des Marstek in Home Assistant anzuzeigen. Ziel ist es, den **Marstek Venus als gezielt steuerbaren Bestandteil eines übergeordneten Energiemanagements** nutzbar zu machen.

In meinem Haus wird der PV-Überschuss nach einer eigenen Prioritätslogik verteilt:

1. **Elektroauto zuerst:** möglichst der gesamte verfügbare PV-Überschuss soll zunächst in das EV fließen.
2. **Warmwasser danach:** lädt das Auto nicht oder ist es voll, wird ein elektrischer Heizstab über Node-RED in mehreren Stufen zwischen **0 und 3000 W** geregelt.
3. **Marstek zuletzt:** erst wenn weder EV noch Warmwasser den Überschuss benötigen, wird der Marstek geladen.

Darüber hinaus verwende ich verschiedene ZigBee Devices um z.B. beim Duschen einen "Boost" anzufordern wenn das warme Wasser knapp wird. Der Marstek selbst dient hauptsächlich als **zeitlicher Energiespeicher**, der überschüssige PV-Energie für die spätere Hauslast – insbesondere nachts – vorhält. Umgekehrt soll er **nicht entladen, um EV oder Heizstab zu versorgen**. Das alles steuere ich seit Jahren mit einem bewährten Node-RED Flow. Wegen dieser komplizierten Logik kann ich bei mir die automatisierte Eigenverbrauchsoptimierung des Marstek nicht verwenden.

Dieses Projekt stellt nur die allgemeingültige **ESPHome-/RS485-/Home-Assistant-Schnittstelle** bereit, die dann z.B. über Automatisierungen oder einfach nur zur Visualisierung in Home Assistant verwendet werden kann. Die konkrete Node-RED-Energiemanagement-Logik ist bewusst **nicht Bestandteil dieses Repositories**, da sie stark von PV-Anlage, Wallbox, Fahrzeug, Warmwasserbereitung und persönlichen Prioritäten abhängt. 

## Hardware / Teileliste

| Bauteil | Zweck | Hinweis | Bezugsquelle |
|---|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, WLAN und lokales Display | ST7789, 135 × 240 px | [Amazon*](https://link.amazon/B0gITBNf5) |
| TTL ↔ RS485 Transceiver-Modul | Modbus/RS485-Schnittstelle | verwendetes Modul mit Pins `3-5V`, `RX-I`, `TX-O`, `RTS`, `GND` sowie `A/B/G` | [Amazon*](https://link.amazon/B0evMwitV) |
| 5-V-Netzteil | feste Versorgung des Controllers | +5 V und GND gehen auf die grüne Trägerplatine; Micro-USB ist nicht die reguläre Versorgung | [Amazon*](https://link.amazon/B047QzbuV) |

\* **Affiliate-Hinweis:** Mit `*` gekennzeichnete Links können Affiliate-Links sein. Wenn du darüber etwas kaufst, kann der Projektbetreiber eine kleine Provision erhalten. Für dich ändert sich der Preis dadurch nicht.

## Verdrahtung

### Gesamtübersicht

Der Marstek selbst versorgt den Controller **nicht**. TTGO und RS485-Transceiver werden aus dem **externen 5-V-Netzteil** versorgt. Vom Marstek kommen nur die drei Leitungen **A, B und G** der RS485-Verbindung.

```text
230 V AC
   │
   ▼
5-V-Netzteil
   │  +5 V / GND
   ▼
grüne Träger-/Lochrasterplatine
   ├──────────────► TTGO T-Display
   └──────────────► RS485-Modul: 3-5V + GND

TTGO T-Display                  RS485-Modul
GPIO27 (TX) ──────────────────► RX-I
GPIO26 (RX) ◄────────────────── TX-O
GPIO25      ──────────────────► RTS
GND         ─────────────────── GND

RS485-Modul                     T568B-Patchkabel      Marstek RJ45
A ────────────────────────────► weiß/orange ────────► Pin 1
B ────────────────────────────► orange ─────────────► Pin 2
G ────────────────────────────► blau ───────────────► Pin 4
```

### Komplette Anschluss-Tabelle

| Von | Nach | Funktion |
|---|---|---|
| externes Netzteil +5 V | grüne Trägerplatine | Versorgung |
| externes Netzteil GND | grüne Trägerplatine | Masse |
| Trägerplatine +5 V | TTGO 5-V-Versorgung | TTGO Versorgung |
| Trägerplatine GND | TTGO GND | TTGO Masse |
| Trägerplatine +5 V | RS485-Modul `3-5V` | Versorgung RS485-Modul |
| Trägerplatine GND | RS485-Modul `GND` | Masse RS485-Modul |
| TTGO GPIO27 (TX) | RS485 `RX-I` | UART vom TTGO zum Transceiver |
| TTGO GPIO26 (RX) | RS485 `TX-O` | UART vom Transceiver zum TTGO |
| TTGO GPIO25 | RS485 `RTS` | Sende-/Empfangsumschaltung |
| RS485 `A` | T568B weiß/orange | Marstek RS485 A, RJ45 Pin 1 |
| RS485 `B` | T568B orange | Marstek RS485 B, RJ45 Pin 2 |
| RS485 `G` | T568B blau | RS485-Bezugspotential, RJ45 Pin 4 |

### Stromversorgung

Das externe Netzteil liefert 5 V auf die grüne Träger-/Lochrasterplatine. Von dort werden sowohl der TTGO als auch das RS485-Modul versorgt.

> **Wichtig:** Vor dem Anschluss Spannung und Polarität prüfen. Eine externe 5-V-Einspeisung und USB nicht gleichzeitig anschließen, wenn nicht sichergestellt ist, dass die konkrete Hardware das unterstützt.

### ESPHome-Pins

Die TTGO-Pins sind zusätzlich durch die aktuelle YAML festgelegt:

```yaml
uart:
  id: uart_rs485
  tx_pin: GPIO27
  rx_pin: GPIO26
  baud_rate: 115200
  data_bits: 8
  parity: NONE
  stop_bits: 1

modbus:
  id: modbus1
  uart_id: uart_rs485
  flow_control_pin: GPIO25
```

Referenzparameter:

```text
Baudrate: 115200
Datenbits: 8
Parity: NONE
Stopbits: 1
Modbus Slave-ID: 1
```

### Patchkabel / RJ45

Im Referenzaufbau wurde ein normales **T568B-Ethernet-Patchkabel** verwendet und an einer Seite abgeschnitten. Der RJ45-Stecker bleibt am Marstek; am offenen Ende werden nur drei Adern verwendet:

| RJ45-Pin | T568B-Farbe | RS485-Modul |
|---:|---|---|
| 1 | weiß/orange | A |
| 2 | orange | B |
| 4 | blau | G |

Am kleinen 3-poligen Steckverbinder im Controller liegen die Adern – **in der Orientierung der Projektfotos** – von oben nach unten als **blau → weiß/orange → orange**.

> **Wichtig:** Nicht blind nach Farben anschließen. Bei einem anderen oder anders belegten Patchkabel die RJ45-Pins mit einem Durchgangsprüfer identifizieren. Entscheidend sind die Pins 1, 2 und 4.

### Innenaufbau

<p align="center"><img src="images/case_open.jpg" alt="Innenansicht des Controllers" width="900"></p>

TTGO, Trägerplatine, Netzteil und RS485-Interface sind gemeinsam im Gehäuse untergebracht.

## Display und die beiden Tasten

<p align="center"><img src="images/display.jpg" alt="TTGO Display im Betrieb" width="900"></p>

Das Display zeigt unter anderem Sollwert (**SET**), Ladezustand (**SOC**), aktuelle Leistung sowie Kommunikations- und WLAN-Status.

> **Experimentell / noch nicht am realen Gerät getestet:** Die Tastenfunktion ist in der YAML implementiert, wurde am gezeigten Aufbau aber noch nicht praktisch verifiziert.

- **Obere Taste (GPIO0):** +100 W je Druck; positive Werte bedeuten Laden.
- **Untere Taste (GPIO35):** −100 W je Druck; negative Werte bedeuten Entladen.
- Bei **0 W** wird die Leistungsausgabe gestoppt.
- Bereich: **−2500 W bis +2500 W**.
- Nur aktiv, wenn **Venus Master (RS485 Enable)** eingeschaltet ist.

## ESPHome Installation

1. Neues Gerät in ESPHome erstellen und **Espressif ESP32 Dev Module** wählen.
2. [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) in ESPHome übernehmen.
3. `secrets.yaml` anhand von [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml) anlegen.
4. YAML validieren und TTGO flashen, z.B. per USB direkt am PC oder am Home-Assistant-Host.
5. Das Gerät wird in Home Assistant unter **Geräte und Dienste** entdeckt und kann dort hinzugefügt werden.

## Marstek-App-Einstellungen / Betriebsmodus

Dieses Projekt steuert den Venus über die **physische RS485-/Modbus-RTU-Schnittstelle** und nicht über die netzwerkbasierte Marstek Local API.

Für den **getesteten V148-Referenzaufbau** gilt:

- **Local API / Open API muss nicht aktiviert sein.** Sie ist eine separate Schnittstelle und wird von diesem Projekt nicht benötigt.
- Den Marstek-Betriebsmodus auf **Manuell** stellen, bevor die externe RS485-Steuerung die Lade-/Entladeleistung übernimmt.
- Während aktiver RS485-Steuerung möglichst keine Betriebsart in der Marstek-App wechseln. Firmwareverhalten kann dabei die externe RS485-Steuerung zurücksetzen oder deaktivieren.
- Die Entscheidung über Laden und Entladen übernimmt dann die eigene Home-Assistant-/Node-RED-Logik über die von diesem Projekt bereitgestellten Entitäten.

### Was ist mit einem Marstek Smart Meter?

Dieser Aufbau ersetzt bewusst die automatische Marstek-Regelung durch eine **eigene externe Regelung**. Ein Marstek Smart Meter kann zwar weiterhin vorhanden sein oder Messwerte liefern, aber **während der erzwungenen RS485-Steuerung sollte man nicht erwarten, dass er die Lade-/Entladeleistung des Speichers automatisch regelt**. Im manuellen/Force-Control-Betrieb muss die externe Automation die Entscheidungen deshalb selbst treffen. Dazu gehört auch, unerwünschte Fälle zu verhindern – etwa dass der Speicher das Elektroauto, einen Heizstab oder einen anderen flexiblen Verbraucher aus dem Akku versorgt.

## Home Assistant

<p align="center"><img src="images/HA%20Screenshot.jpg" alt="ESPHome-Gerät in Home Assistant" width="900"></p>

Bereitgestellt werden unter anderem SoC, AC-Leistung, geglättete Leistung, Temperaturwerte, maximale Lade-/Entladeleistung, Modbus-Status, Lade-/Entlade-Sollwerte, RS485 Master, NOT-AUS, OTA Quiet Mode, Restart und WiFi RSSI.

## Warum gibt es den OTA Quiet Mode?

Bei schwachem WLAN soll der ESP32 möglichst viel Reserve für Funk und Kommunikation haben. Das Display verursacht sowohl CPU-/SPI-Last durch das Rendern als auch Stromverbrauch durch die Hintergrundbeleuchtung. Der Quiet Mode schaltet in der aktuellen Konfiguration die Hintergrundbeleuchtung aus und reduziert damit die elektrische Last; das Rendering läuft derzeit weiter. Die Marstek-Steuerlogik bleibt aktiv.

## Steuerregister

| Register | Bedeutung | verwendete Werte |
|---:|---|---|
| `42000` | RS485 Remote Control | `21930` = an, `21947` = aus |
| `42010` | Force Mode | `0` Standby, `1` Laden, `2` Entladen |
| `42020` | erzwungene Ladeleistung | 0…2500 W |
| `42021` | erzwungene Entladeleistung | 0…2500 W |
| `42011` | Charge-to-SoC | % |
| `44002` | maximale Ladeleistung | Readback |
| `44003` | maximale Entladeleistung | Readback |

### Firmware-Hinweis

Marstek-Firmware kann die RS485-Fernsteuerung wieder deaktivieren. Deshalb wird der RS485-Control-Code im Heartbeat ausgegeben:

```text
Heartbeat: ... req=-700 meas=797 rs485=21930 ctrl=2 dis_reg=700 ...
```

Wenn Sollwert und Force Mode korrekt aussehen, aber der Speicher trotzdem 0 W liefert, zuerst `42000` bzw. `rs485` prüfen.

## Was die Firmware zusätzlich macht

Die YAML enthält schnelle/langsame Modbus-Abfragen, mehrstufige Plausibilitäts- und Anti-Glitch-Filter, Medianfilter, SoC-Sprungfilter, Request/Ack-Tracking, Underdelivery-Diagnose, Modbus-Freshness-Überwachung, Recovery/Watchdog, Software-NOT-AUS, TFT-Anzeige und den RS485-Control-Code im Heartbeat.

Eine ausführliche Erklärung steht in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## 3D-gedrucktes Gehäuse

- [`Body.stl`](enclosure/STL/Body.stl) – Gehäusekörper
- [`Cover.stl`](enclosure/STL/Cover.stl) – Frontdeckel mit Display-, Tasten- und Lüftungsöffnungen

## Credits / Vorarbeiten

Dieses Projekt baut auf Arbeit aus der Marstek-Community auf. Besonders wichtig waren **ViperRNMC – marstek_venus_modbus** als Referenz für Marstek-Venus-Modbus-Register und **Superduper1969 – MarstekVenus-LilygoRS485** als ESPHome/LILYGO-RS485-Referenz und Inspiration.

Ein Vergleich der aktuellen YAML ergab **keine wesentliche wortgleiche Übernahme der eigentlichen Anwendungslogik** aus einem der beiden Projekte. Gemeinsam sind vor allem ESPHome-/Modbus-Strukturen, öffentliche Registeradressen/Befehlswerte und übliche RS485-Konzepte. Filterung, Watchdog/Recovery, TFT-Anzeige, Request/Ack- und Diagnoselogik dieses Repositories sind eigenständige Implementierungen.

Details zu Herkunft und Lizenzen stehen in [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).

## Lizenz

Dieses Repository steht unter der **MIT-Lizenz**. Siehe [`LICENSE`](LICENSE).

## Danke sagen

Wenn dir dieses Projekt geholfen hat oder du es erfolgreich für deinen eigenen Aufbau einsetzen konntest, freue ich mich über ein kleines Dankeschön. Dafür kannst du den **Sponsor**-Button oben im Repository nutzen oder mir direkt einen Kaffee über PayPal spendieren.

<a href="https://paypal.me/toor0001"><img src="assets/paypal-support-de.svg" alt="Spendier mir einen Kaffee via PayPal" width="430"></a>

## Hinweis zur Entwicklung

Dieses Projekt ist im Rahmen eines kollaborativen **Vibe-Coding-Workflows** mit **ChatGPT** und **OpenAI Codex** entstanden. Beide Tools wurden für Code-Erstellung, Reviews, Fehlersuche und Dokumentation eingesetzt.

Hardwareaufbau, Integrationsentscheidungen, praktische Tests und die abschließende Verantwortung für das Projekt liegen beim Projektbetreiber.

## Sicherheit

Diese Software kann Lade- und Entladeleistungen eines Batteriespeichers vorgeben. Nutze sie nur, wenn du deine Installation und die zulässigen Grenzen verstehst. Die Schutzfunktionen des Herstellers sollten aktiv bleiben; Änderungen immer zuerst mit kleinen Leistungen testen.

Dieses Projekt ist ein unabhängiges Community-Projekt und **nicht mit Marstek, LILYGO, ESPHome oder Home Assistant verbunden oder von diesen unterstützt**.
