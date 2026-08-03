# Marstek Venus ESPHome HA RS485

Lokale Steuerung und Überwachung eines **Marstek Venus** Speichers mit **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** und **Home Assistant**.

[🇬🇧 English README](README.md)

Der ESP32 spricht direkt per RS485 mit dem Venus. Für die eigentliche Steuerung ist keine Cloud-Verbindung nötig. Das TTGO-Display zeigt die wichtigsten Informationen direkt am Gerät, während Home Assistant Sensoren, Diagnosewerte sowie Lade- und Entlade-Sollwerte bereitstellt.

> **Status:** funktionierendes Projekt / Referenzaufbau. Getestet mit einem Marstek Venus E Gen3 und Firmware V148. Firmwareupdates von Marstek können das Modbus-Verhalten verändern; deshalb nach Updates die Steuerung immer erneut prüfen.

## Der fertige Aufbau

<p align="center"><img src="images/wall_setup.jpg" alt="Marstek Venus mit TTGO-RS485-Steuerung" width="900"></p>

Der kleine TTGO-Controller sitzt direkt neben dem Marstek Venus und übernimmt die lokale RS485-Kommunikation.

<p align="center"><img src="images/case.jpg" alt="3D-gedrucktes TTGO Gehäuse" width="900"></p>

## Warum dieses Projekt?

- direkte **RS485 / Modbus RTU** Verbindung zum Speicher
- **ESPHome** Firmware
- native **Home Assistant** Entitäten
- **TTGO/LILYGO Display** mit SoC, Leistung, WLAN- und Modbus-Status
- manuelle Lade-/Entlade-Sollwerte
- Watchdog und Modbus-Recovery
- Plausibilitätsfilter für SoC und AC-Leistung
- Diagnose von Soll- gegen Ist-Leistung
- experimentelle lokale Bedienung über die beiden TTGO-Tasten
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
| 3D-gedrucktes Gehäuse | mechanischer Schutz | Body + Lid als STL enthalten | [STL-Ordner](enclosure/STL/) |

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

Mindestens die differentiellen Leitungen **A** und **B** werden verbunden. Je nach Modul bzw. Aufbau kann zusätzlich GND sinnvoll oder erforderlich sein. Die Bezeichnungen A/B sind zwischen RS485-Modul-Herstellern nicht immer einheitlich; bei komplett ausbleibender Kommunikation deshalb auch vertauschtes A/B prüfen.

```text
Baudrate: 115200
Datenbits: 8
Parity: NONE
Stopbits: 1
Modbus Slave-ID: 1
```

### Innenaufbau

<p align="center"><img src="images/case_open.jpg" alt="Innenansicht des Controllers" width="900"></p>

Das TTGO T-Display und das RS485-Interface sind gemeinsam im Gehäuse untergebracht. Die Kabel werden seitlich über Kabelverschraubungen herausgeführt.

## Display und die beiden Tasten

<p align="center"><img src="images/display.jpg" alt="TTGO Display im Betrieb" width="900"></p>

Das Display zeigt unter anderem den aktuellen Sollwert (**SET**), den Ladezustand (**SOC**), die aktuelle Leistung sowie Kommunikations- und WLAN-Status.

> **Experimentell / noch nicht am realen Gerät getestet:** Die folgende Tastenfunktion ist in der aktuellen YAML implementiert, wurde am gezeigten Aufbau aber noch nicht praktisch verifiziert. Bitte zunächst nur mit kleinen Sollwerten testen.

Die vorgesehene Funktion der beiden kleinen TTGO-Taster ist:

- **Linke Taste (GPIO0):** erhöht den Sollwert bei jedem Tastendruck um **100 W**. Positive Sollwerte bedeuten **Laden**.
- **Rechte Taste (GPIO35):** verringert den Sollwert bei jedem Tastendruck um **100 W**. Negative Sollwerte bedeuten **Entladen**.
- Bei **0 W** wird die Leistungsausgabe gestoppt.
- Der Bereich ist in der Firmware auf **−2500 W bis +2500 W** begrenzt.
- Die Tastensteuerung wird nur verarbeitet, wenn **Venus Master (RS485 Enable)** aktiv ist.

Beispiel gemäß aktueller Implementierung: Ausgehend von 0 W würde dreimal links **+300 W Laden** auswählen. Dreimal rechts würde wieder zu 0 W führen; weitere Tastendrücke nach rechts würden in den Entladebereich wechseln.

## ESPHome Installation

1. [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) in das ESPHome-Konfigurationsverzeichnis kopieren.
2. `secrets.yaml` anhand von [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml) anlegen bzw. ergänzen.
3. UART- und RS485-Pins mit der eigenen Hardware abgleichen.
4. Konfiguration in ESPHome validieren.
5. TTGO flashen.
6. ESPHome-Gerät in Home Assistant hinzufügen.
7. Vor Automationen zuerst prüfen, ob **Venus Modbus OK** aktiv ist.

## Home Assistant

<p align="center"><img src="images/HA%20Screenshot.jpg" alt="ESPHome-Gerät in Home Assistant" width="900"></p>

Unter anderem werden bereitgestellt: Venus SOC, AC Power, Stable Power, Max Charge/Discharge Power, Temperaturen, Modbus OK, Lade-/Entlade-Sollwerte, RS485 Master, NOT-AUS, OTA Quiet Mode, Restart und WiFi RSSI.

## Warum gibt es den OTA Quiet Mode?

Der TTGO kann an einem Ort mit schwachem WLAN-Empfang betrieben werden. Gerade bei OTA-Updates oder bei der Fehlersuche möchten wir dem ESP32 möglichst viel Reserve für WLAN und Kommunikation geben.

Das Display erzeugt zusätzliche Last auf zwei Ebenen: Das regelmäßige Zeichnen und Aktualisieren benötigt Rechenzeit und SPI-Kommunikation; die Hintergrundbeleuchtung benötigt zusätzlich elektrische Leistung. Der **TTGO OTA Quiet Mode** reduziert unnötige Display-Last. In der aktuellen Konfiguration wird die Hintergrundbeleuchtung abgeschaltet. Dadurch sinkt der Strombedarf des Displays und die Versorgung hat mehr Reserve für ESP32 und WLAN. Das regelmäßige Rendern der Anzeige läuft derzeit weiter.

Der Quiet Mode schaltet **nicht** die Marstek-Steuerlogik aus. Er ist ein Service-/Diagnosemodus für den TTGO selbst.

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

Marstek-Firmware kann die RS485-Fernsteuerung wieder deaktivieren. Deshalb wird der RS485-Control-Code im Heartbeat ausgegeben. Ein gesunder Entlade-Heartbeat kann z. B. so aussehen:

```text
Heartbeat: ... req=-700 meas=797 rs485=21930 ctrl=2 dis_reg=700 ...
```

Wenn Sollwert und Force Mode korrekt aussehen, aber der Speicher trotzdem 0 W liefert, zuerst `42000` bzw. `rs485` prüfen.

## Was die Firmware zusätzlich macht

Die YAML enthält getrennte schnelle/langsame Modbus-Abfragen, Plausibilitäts- und Medianfilter, SoC-Sprungfilter, Request/Ack-Tracking, Underdelivery-Diagnose, Modbus-Freshness-Überwachung, Recovery/Watchdog, Software-NOT-AUS, TFT-Anzeige, experimentelle lokale Tastensteuerung und den RS485-Control-Code im Heartbeat.

Eine ausführliche Erklärung steht in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## 3D-gedrucktes Gehäuse

Das Gehäuse besteht aus zwei druckbaren Teilen:

- [`Body.stl`](enclosure/STL/Body.stl) – Gehäusekörper
- [`Lid.stl`](enclosure/STL/Lid.stl) – Frontdeckel mit Display-, Tasten- und Lüftungsöffnungen

Die Dateien können direkt aus dem Repository heruntergeladen und im Slicer geöffnet werden. Das gezeigte Gehäuse nimmt TTGO, RS485-Interface und Verdrahtung gemeinsam auf.

## Credits / Vorarbeiten

Dieses Projekt baut auf Arbeit aus der Marstek-Community auf. Besonders wichtig waren **ViperRNMC – marstek_venus_modbus** als Referenz für Marstek-Venus-Modbus-Register und **Superduper1969 – MarstekVenus-LilygoRS485** als ESPHome/LILYGO-RS485-Referenz und Inspiration.

## Projekt unterstützen

Wenn dir das Projekt geholfen hat und du die Weiterentwicklung unterstützen möchtest:

<a href="https://paypal.me/toor0001"><img src="assets/paypal-support-de.svg" alt="Spendiere mir einen Kaffee via PayPal" width="430"></a>

## Hinweis zur Entwicklung

Dieses Projekt ist im Rahmen eines kollaborativen **Vibe-Coding-Workflows** mit **ChatGPT** und **OpenAI Codex** entstanden. Beide Tools wurden für Code-Erstellung, Reviews, Fehlersuche und Dokumentation eingesetzt.

Hardwareaufbau, Integrationsentscheidungen, praktische Tests und die abschließende Verantwortung für das Projekt liegen beim Projektbetreiber.

## Sicherheit

Diese Software kann Lade- und Entladeleistungen eines Batteriespeichers vorgeben. Nutze sie nur, wenn du deine Installation und die zulässigen Grenzen verstehst. Die Schutzfunktionen des Herstellers sollten aktiv bleiben; Änderungen immer zuerst mit kleinen Leistungen testen.

Dieses Projekt ist ein unabhängiges Community-Projekt und **nicht mit Marstek, LILYGO, ESPHome oder Home Assistant verbunden oder von diesen unterstützt**.
