# Hardware & Verdrahtung

## Referenz-Hardware

Dieses Projekt wurde für einen direkten RS485-Zugriff auf einen Marstek Venus mit einem ESP32 aufgebaut.

| Bauteil | Zweck | Hinweis |
|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, WLAN, Home Assistant, Display | ST7789, 135 × 240 px |
| TTL ↔ RS485 Transceiver-Modul | elektrische RS485-Schnittstelle | Logikseite muss zum ESP32 passen; 3,3-V-Kompatibilität prüfen |
| USB-Netzteil + USB-Kabel | Versorgung des TTGO | saubere, stabile 5-V-Versorgung verwenden |
| RS485-Leitung | Verbindung zum Venus | verdrilltes A/B-Paar empfohlen |
| passender Stecker für den Marstek-RS485-Port | Anschluss am Speicher | Pinbelegung vor Anschluss am eigenen Gerät prüfen |
| optional: 3D-gedrucktes Gehäuse | mechanischer Schutz | STL-Ordner ist vorbereitet |

> **Wichtig:** RS485-Module sehen oft sehr ähnlich aus, können aber unterschiedliche Versorgungsspannungen, Pegelwandler oder Pinbelegungen besitzen. Deshalb nicht allein nach Platinenfarbe oder Modulform verdrahten.

## ESP32 ↔ RS485-Modul

Die aktuelle ESPHome-Konfiguration verwendet:

| TTGO / ESP32 | RS485-Modul | Funktion |
|---|---|---|
| GPIO27 | DI / TX | UART TX vom ESP32 zum Transceiver |
| GPIO26 | RO / RX | UART RX vom Transceiver zum ESP32 |
| GPIO25 | DE + /RE | Sende-/Empfangsumschaltung |
| GND | GND | gemeinsame Masse |
| VCC | VCC | entsprechend der Spezifikation des verwendeten Moduls |

Im YAML:

```yaml
uart:
  id: uart_rs485
  tx_pin: GPIO27
  rx_pin: GPIO26
  baud_rate: 115200

modbus:
  id: modbus1
  uart_id: uart_rs485
  flow_control_pin: GPIO25
```

## RS485 ↔ Marstek Venus

Am Bus werden mindestens die beiden differentiellen Leitungen **A** und **B** verbunden. Je nach Aufbau kann zusätzlich GND sinnvoll bzw. erforderlich sein.

Leider ist die Bezeichnung A/B zwischen Herstellern nicht vollständig einheitlich. Wenn überhaupt keine Antworten kommen, obwohl UART, Baudrate und Slave-ID stimmen, ist eine vertauschte A/B-Leitung ein typischer Prüfpunkt.

### Busparameter des Referenzaufbaus

```text
Baudrate: 115200
Datenbits: 8
Parity: NONE
Stopbits: 1
Modbus-Adresse / Slave-ID: 1
```

## TTGO T-Display Pins

Das Display selbst wird von ESPHome über die bekannten T-Display-Pins angesteuert:

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

Dadurch bleibt der externe RS485-Anschluss auf GPIO26/27/25 frei von Displayfunktionen.

## Vor dem ersten Einschalten

1. Versorgungsspannung und Logikpegel des RS485-Moduls prüfen.
2. GND-Verbindung prüfen.
3. A/B-Verbindung prüfen.
4. Erst ohne automatische Lade-/Entladeregelung testen.
5. In ESPHome kontrollieren, ob `Venus Modbus OK` aktiv wird.
6. SoC und Leistung auf plausible Werte prüfen.
7. Erst danach kleine Lade-/Entladesollwerte ausprobieren, z. B. 100–300 W.

## Gehäuse

Die vorgesehenen 3D-Druckdateien liegen unter `enclosure/STL/`. Die STL-Dateien können später ergänzt oder ausgetauscht werden, ohne die restliche Projektstruktur zu ändern.
