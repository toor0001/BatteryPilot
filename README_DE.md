# Marstek Venus ESPHome HA RS485

Lokale Steuerung und Überwachung eines **Marstek Venus** Speichers mit **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** und **Home Assistant**.

[🇬🇧 English README](README.md)

Der ESP32 spricht direkt per RS485 mit dem Venus. Für die eigentliche Steuerung ist keine Cloud-Verbindung nötig. Das TTGO-Display zeigt die wichtigsten Informationen direkt am Gerät, während Home Assistant Sensoren, Diagnosewerte sowie Lade- und Entlade-Sollwerte bereitstellt.

> **Status:** funktionierendes Projekt / Referenzaufbau. Getestet mit einem Marstek Venus E Gen3 und Firmware V148. Firmwareupdates von Marstek können das Modbus-Verhalten verändern; deshalb nach Updates die Steuerung immer erneut prüfen.

## Warum dieses Projekt?

- direkte **RS485 / Modbus RTU** Verbindung zum Speicher
- **ESPHome** Firmware
- native **Home Assistant** Entitäten
- **TTGO/LILYGO Display** mit SoC, Leistung, WLAN- und Modbus-Status
- manuelle Lade-/Entlade-Sollwerte
- Watchdog und Modbus-Recovery
- Plausibilitätsfilter für SoC und AC-Leistung
- Diagnose von Soll- gegen Ist-Leistung
- lokale Bedienung über die beiden TTGO-Tasten
- optionales 3D-gedrucktes Gehäuse
- lokale Steuerung ohne Cloud-Zwang

## Hardware / Teileliste

| Bauteil | Zweck | Hinweis | Bezugsquelle |
|---|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, WLAN und lokales Display | ST7789, 135 × 240 px | [Amazon*](https://link.amazon/B0gITBNf5) |
| TTL ↔ RS485 Transceiver-Modul | physische Modbus/RS485-Schnittstelle | 3,3-V-Kompatibilität prüfen | [Amazon*](https://link.amazon/B0evMwitV) |
| USB-Netzteil + USB-Kabel | Versorgung des TTGO | stabile 5-V-Versorgung empfohlen | [Amazon*](https://link.amazon/B0cxu0tlI) |
| RS485-Leitung | Verbindung zum Venus | verdrilltes A/B-Paar empfohlen | Link folgt |
| passender Stecker für den Marstek-RS485-Port | Anschluss am Speicher | Pinbelegung am eigenen Gerät prüfen | Link folgt |
| 3D-gedrucktes Gehäuse | mechanischer Schutz | STL-Dateien unter `enclosure/STL/` | enthalten |

\* **Affiliate-Hinweis:** Mit `*` gekennzeichnete Links können Affiliate-Links sein. Wenn du darüber etwas kaufst, kann der Projektbetreiber eine kleine Provision erhalten. Für dich ändert sich der Preis dadurch nicht.

## Verdrahtung

### ESP32 ↔ RS485-Modul

| TTGO / ESP32 | RS485-Modul | Funktion |
|---|---|---|
| GPIO27 | DI / TX | UART TX |
| GPIO26 | RO / RX | UART RX |
| GPIO25 | DE + /RE | RS485 Sende-/Empfangsumschaltung |
| GND | GND | gemeinsame Masse |
| VCC | VCC | passend zum verwendeten RS485-Modul |

Im ESPHome-YAML:

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

### RS485 ↔ Marstek Venus

Mindestens die beiden differentiellen Leitungen **A** und **B** werden verbunden. Je nach Modul bzw. Aufbau kann zusätzlich GND sinnvoll oder erforderlich sein.

> Die Bezeichnungen A/B sind zwischen RS485-Modul-Herstellern leider nicht immer einheitlich. Wenn überhaupt keine Kommunikation zustande kommt, ist vertauschtes A/B einer der ersten Prüfpunkte.

Referenzparameter:

```text
Baudrate: 115200
Datenbits: 8
Parity: NONE
Stopbits: 1
Modbus Slave-ID: 1
```

### TTGO T-Display Pins

| Funktion | GPIO |
|---|---:|
| SPI CLK | 18 |
| SPI MOSI | 19 |
| TFT CS | 5 |
| TFT DC | 16 |
| TFT RESET | 23 |
| Backlight PWM | 4 |
| linke Taste | 0 |
| rechte Taste | 35 |

## ESPHome Installation

1. [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) in das ESPHome-Konfigurationsverzeichnis kopieren.
2. `secrets.yaml` anhand von [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml) anlegen bzw. ergänzen.
3. UART- und RS485-Pins mit der eigenen Hardware abgleichen.
4. Konfiguration in ESPHome validieren.
5. TTGO flashen.
6. ESPHome-Gerät in Home Assistant hinzufügen.
7. Vor Automationen zuerst prüfen, ob **Venus Modbus OK** aktiv ist.

## Home-Assistant-Entitäten

Unter anderem werden bereitgestellt:

- Venus SOC
- Venus AC Power
- Venus AC Power W Stable
- Venus Max Charge Power
- Venus Max Discharge Power
- Venus Charge To SoC
- Zell-/Innentemperaturen
- Venus Modbus OK
- Venus Set Charge Power W
- Venus Set Discharge Power W
- Venus Master (RS485 Enable)
- Venus NOT-AUS
- TTGO OTA Quiet Mode
- TTGO Restart
- TTGO WiFi RSSI

## Warum gibt es den OTA Quiet Mode?

Der TTGO kann an einem Ort mit schwachem WLAN-Empfang betrieben werden. Gerade bei OTA-Updates oder bei der Fehlersuche möchten wir dem ESP32 möglichst viel Reserve für WLAN und die eigentliche Kommunikation geben.

Das Display verursacht dabei zwei Arten zusätzlicher Last: Das regelmäßige Zeichnen und Aktualisieren der Anzeige benötigt Rechenzeit und SPI-Kommunikation, während die Hintergrundbeleuchtung zusätzlich Strom aus der Versorgung benötigt. Der **TTGO OTA Quiet Mode** dient deshalb dazu, das Display während solcher Situationen möglichst aus dem Weg zu nehmen und unnötige Last zu reduzieren. In der aktuellen Konfiguration wird dabei die Hintergrundbeleuchtung abgeschaltet; dadurch sinkt insbesondere der Strombedarf des Displays und die Versorgung hat mehr Reserve für ESP32 und WLAN.

Der Modus ist besonders sinnvoll wenn:

- der TTGO schlechten WLAN-Empfang hat,
- ein OTA-Update instabil läuft,
- man während der Fehlersuche möglichst wenig unnötige Display-Last haben möchte.

Wichtig: Der Quiet Mode schaltet **nicht** die Marstek-Steuerlogik aus. Er ist ein Service-/Diagnosemodus für den TTGO selbst.

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

Marstek-Firmware kann die RS485-Fernsteuerung wieder deaktivieren, z. B. wenn Betriebsarten über die App oder andere Modbus-Register verändert werden. Deshalb wird der RS485-Control-Code im Heartbeat mit ausgegeben.

Ein gesunder Entlade-Heartbeat kann z. B. so aussehen:

```text
Heartbeat: ... req=-700 meas=797 rs485=21930 ctrl=2 dis_reg=700 ...
```

Wenn Sollwert und Force Mode korrekt aussehen, aber der Speicher trotzdem 0 W liefert, zuerst `42000` bzw. `rs485` prüfen.

## Was die Firmware zusätzlich macht

Die YAML enthält unter anderem:

- getrennte schnelle und langsame Modbus-Abfragen
- Plausibilitätsprüfung der AC-Leistung
- Median-of-3-Filter
- SoC-Sprungfilter
- Request/Ack-Tracking
- Underdelivery-Diagnose
- Modbus-Freshness-Überwachung
- einmaligen RS485-Recovery-Versuch bei ausbleibenden Daten
- TTGO-Neustart bei dauerhaftem Kommunikationsausfall
- Software-NOT-AUS
- lokale TFT-Anzeige
- Bedienung über beide TTGO-Tasten
- RS485-Control-Code im Heartbeat

Eine ausführliche Erklärung des Codes steht in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## Vor dem ersten Einsatz

1. Versorgungsspannung und Logikpegel des RS485-Moduls prüfen.
2. GND und A/B kontrollieren.
3. Zunächst ohne automatische Lade-/Entladeregelung testen.
4. `Venus Modbus OK` prüfen.
5. SoC und Leistung auf Plausibilität prüfen.
6. Erst kleine Sollwerte testen, z. B. 100–300 W.
7. Erst danach unbeaufsichtigte Automationen aktivieren.

## 3D-gedrucktes Gehäuse

Die STL-Dateien liegen unter [`enclosure/STL/`](enclosure/STL/).

## Credits / Vorarbeiten

Dieses Projekt baut auf Arbeit aus der Marstek-Community auf. Besonders wichtig waren:

- **ViperRNMC – marstek_venus_modbus** – wichtige Referenz für Marstek-Venus-Modbus-Register und das Verhalten der Home-Assistant-Anbindung.
- **Superduper1969 – MarstekVenus-LilygoRS485** – ESPHome/LILYGO-RS485-Referenz und Inspiration für eine direkte ESP-basierte Marstek-Anbindung.

## Projekt unterstützen

Wenn dir das Projekt geholfen hat und du die Weiterentwicklung unterstützen möchtest:

<a href="https://paypal.me/toor0001"><img src="assets/paypal-support-de.svg" alt="Spendiere mir einen Kaffee via PayPal" width="430"></a>

## Sicherheit

Diese Software kann Lade- und Entladeleistungen eines Batteriespeichers vorgeben. Nutze sie nur, wenn du deine Installation und die zulässigen Grenzen verstehst. Die Schutzfunktionen des Herstellers sollten aktiv bleiben; Änderungen immer zuerst mit kleinen Leistungen testen.

Dieses Projekt ist ein unabhängiges Community-Projekt und **nicht mit Marstek, LILYGO, ESPHome oder Home Assistant verbunden oder von diesen unterstützt**.
