# Code erklärt

Die ESPHome-Konfiguration ist mehr als eine reine Registerliste. Sie übernimmt Kommunikation, Filterung, Steuerung, Diagnose, Watchdog und die lokale Anzeige auf dem TTGO.

## 1. Boot-Verhalten

Beim Start wird der RS485-Master zunächst deaktiviert. Erst nach einer kurzen Wartezeit und erfolgreicher WLAN-Verbindung wird die Fernsteuerung eingeschaltet.

Das verhindert, dass beim Booten sofort alte Sollwerte auf den Speicher geschrieben werden.

Wichtige Zustände:

- `master_enabled` – lokale Freigabe der RS485-Steuerung
- `last_rx_ms` – Zeitpunkt der letzten gültigen Modbus-Antwort
- `boot_ms` – Startzeit für Watchdog-Grace-Perioden
- `wd_recover_attempted` – verhindert endlose Recovery-Schleifen

## 2. Zwei Modbus-Controller

Die Konfiguration verwendet zwei Polling-Geschwindigkeiten:

```text
venus_fast  -> schnelle Werte, z. B. Leistung und SoC
venus_slow  -> Limits, Temperaturen und Konfigurationswerte
```

So bleiben Leistung und SoC reaktionsschnell, ohne den RS485-Bus mit allen langsamen Registern permanent zu belasten.

## 3. AC-Power-Filter

Der Rohwert der AC-Leistung kommt aus Register `30006`.

Der Code schützt gegen typische Ausreißer:

- unplausibel große Werte werden verworfen
- einzelne kurze 0-W-Ausreißer werden nicht sofort übernommen
- die letzten drei Werte werden über einen Median-of-3-Filter geglättet

Das Ergebnis ist `Venus AC Power W Stable` und wird für Anzeige und Diagnose verwendet.

## 4. SoC-Filter

Der SoC wird nicht blind übernommen. Große Sprünge innerhalb eines kurzen Zeitfensters werden verworfen.

Das hilft gegen einzelne fehlerhafte Modbus-Lesungen. Dadurch kann der veröffentlichte `Venus SOC` kurzfristig hinter dem Rohwert liegen – das ist beabsichtigt.

## 5. Steuerregister

Für die Gen3-Steuerung werden die aus der Community bekannten Register verwendet:

| Register | Bedeutung |
|---:|---|
| 42000 | RS485 Remote Control |
| 42010 | Force Mode |
| 42020 | Ladeleistung |
| 42021 | Entladeleistung |

### RS485 Remote Control

```text
21930 = Remote Control aktivieren
21947 = Remote Control deaktivieren
```

### Force Mode

```text
0 = Standby
1 = Charge
2 = Discharge
```

Die Home-Assistant-Entitäten `Venus Set Charge Power W` und `Venus Set Discharge Power W` setzen diese Register über ESPHome-Scripts.

## 6. Master-Switch

`Venus Master (RS485 Enable)` ist die zentrale Freigabe.

Beim Einschalten:

```text
42000 <- 21930
master_enabled = true
```

Beim Ausschalten:

1. Force Mode auf 0
2. Ladeleistung auf 0
3. Entladeleistung auf 0
4. `42000 <- 21947`

Dadurch wird die Fernsteuerung sauber verlassen.

## 7. NOT-AUS

Der Button `Venus NOT-AUS` setzt die Ausgabe sofort auf 0, deaktiviert RS485 Remote Control und setzt lokale Sollwerte zurück.

Das ist kein Ersatz für die Schutzfunktionen des Speichers oder eine elektrische Not-Aus-Einrichtung, sondern ein Software-Stopp für dieses Projekt.

## 8. Request-Tracking und Underdelivery

Der Code merkt sich den zuletzt angeforderten Sollwert und vergleicht ihn mit der gemessenen Leistung.

Beispiel:

```text
req=-700
meas=797
rs485=21930
ctrl=2
dis_reg=700
```

Damit kann man schnell unterscheiden:

- Sollwert kommt nicht am ESP an
- Force Mode ist falsch
- RS485 Remote Control wurde deaktiviert
- Register enthält den Sollwert, aber der Speicher setzt ihn nicht um

Die `UNDERDELIVER`-Diagnose meldet auffällige Abweichungen nach mehreren Sekunden.

## 9. Heartbeat

Alle 30 Sekunden schreibt der TTGO eine kompakte Diagnosezeile ins Log.

Besonders wichtig sind:

- `modbus_ok`
- `rx_age_s`
- `soc`
- `ac`
- `req`
- `meas`
- `rs485`
- `ctrl`
- `dis_reg`
- `max_dis`

Nach Firmwareupdates des Marstek ist der Heartbeat ein sehr hilfreicher erster Check.

## 10. Modbus-Watchdog

Wenn für längere Zeit keine Modbus-Daten mehr empfangen werden:

1. nach >90 s wird der RS485-Master einmal aus- und wieder eingeschaltet
2. bleibt der Bus weiterhin tot, wird später der TTGO neu gestartet

Vor dem Recovery werden Sollwerte auf 0 gesetzt.

## 11. Display

Das TTGO-Display zeigt lokal:

- Modbus OK / Fehler
- WLAN-Qualität
- aktuellen Sollwert
- SoC
- aktuelle Lade- oder Entladeleistung
- farbliche Kennzeichnung der Richtung

So ist der Zustand des Speichers auch ohne Home Assistant direkt am Gerät sichtbar.

## 12. Tasten

Die beiden Tasten am T-Display verändern den lokalen Setpoint schrittweise und rufen anschließend dieselbe Modbus-Steuerlogik auf.

Dadurch lässt sich die Verbindung auch unabhängig von Home Assistant testen.

## 13. Home Assistant / Node-RED

ESPHome veröffentlicht die relevanten Sensoren und Number-Entities nativ in Home Assistant. Eine Regelung kann deshalb entweder mit HA-Automationen oder Node-RED aufgebaut werden.

Die eigentliche Energie-Logik – z. B. Null-Einspeisung, Nachtprofil, Mindest-SoC oder PV-Überschuss – gehört bewusst nicht fest in dieses Repo. Das Projekt stellt die robuste lokale Schnittstelle zum Marstek bereit.

## 14. Firmware-Updates des Marstek

Nach einem Marstek-Firmwareupdate zuerst manuell prüfen:

1. Modbus-Lesen funktioniert
2. `42000` liefert bei aktivem Master `21930`
3. `42010` lässt sich auf 1 bzw. 2 setzen
4. `42020/42021` übernehmen kleine Sollwerte
5. die reale Leistung folgt dem Sollwert

Erst danach automatische Regelungen wieder aktivieren.

## Credits

Die Registerbelegung und viele Erkenntnisse rund um Marstek Modbus stammen aus Community-Arbeit. Besonders wichtige Referenzen für dieses Projekt sind:

- ViperRNMC: https://github.com/ViperRNMC/marstek_venus_modbus
- Superduper1969: https://github.com/Superduper1969/MarstekVenus-LilygoRS485
