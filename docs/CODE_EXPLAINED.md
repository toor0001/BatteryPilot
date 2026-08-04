# Code erklärt

Die ESPHome-Konfiguration ist mehr als eine reine Registerliste. Sie übernimmt Kommunikation, Filterung, Steuerung, Diagnose, Watchdog und die lokale Anzeige auf dem TTGO.

## 1. Boot-Verhalten

Beim Start wird der RS485-Master zunächst deaktiviert. Erst nach einer kurzen Wartezeit und erfolgreicher WLAN-Verbindung wird die Fernsteuerung eingeschaltet. Das verhindert, dass beim Booten sofort alte Sollwerte auf den Speicher geschrieben werden.

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

## 3. Mehrstufige Messwertaufbereitung / Anti-Glitch-Pipeline

Die Rohwerte des Venus werden nicht ungeprüft direkt an Home Assistant oder das Display weitergereicht. Gerade bei serieller Kommunikation können einzelne falsche oder kurzzeitig unplausible Werte auftreten. Deshalb durchläuft insbesondere die AC-Leistung mehrere Schutzstufen. DAs funktioniert eigentlich sehr zuverlässig.

Der Datenweg sieht vereinfacht so aus:

```text
Modbus RAW
   ↓
Hard-Reject
   ↓
Plausibilitätsprüfung
   ↓
Zero-Glitch-Bestätigung
   ↓
Median-of-3
   ↓
Stable Power
   ↓
Home Assistant / Display / Diagnose
```

Wichtig ist: Die Filter erfüllen unterschiedliche Aufgaben. Ein einzelner Medianfilter würde zum Beispiel einen echten Kommunikationsfehler mit sehr großem Zahlenwert nicht grundsätzlich verhindern. Umgekehrt würde eine reine Grenzwertprüfung kurze 0-W-Glitches nicht zuverlässig abfangen.

### 3.1 AC-Power-Rohwert

Die rohe AC-Leistung wird aus Register `30006` gelesen und zunächst intern als `Venus AC Power raw (fast)` verarbeitet. Jede gültige eingehende Messung aktualisiert außerdem `last_rx_ms`. Damit dient derselbe Datenstrom gleichzeitig als Lebenszeichen für die Modbus-Kommunikation.

### 3.2 Hard-Reject ungültiger Werte

Zuerst werden offensichtlich kaputte Werte verworfen:

```text
nicht endlich / NaN / Inf  -> verwerfen
|Wert| > 20000 W           -> verwerfen
```

Die 20-kW-Grenze ist eine harte Schutzgrenze gegen völlig falsche Registerwerte oder Decode-Ausreißer.

### 3.3 Plausibilitätsgrenze

Danach folgt eine engere Grenze für den realistisch erwarteten Betrieb:

```text
|Wert| > 6500 W -> verwerfen
```

Der Referenzspeicher arbeitet deutlich unterhalb dieser Leistung. Dadurch gelangen unrealistische Werte gar nicht erst in die weitere Verarbeitung. Die beiden Grenzen sind absichtlich getrennt:

- `ABS_MAX = 20000 W` schützt gegen offensichtlich defekte Daten
- `PLAUS_MAX = 6500 W` schützt gegen Werte, die zwar technisch darstellbar, für diesen Aufbau aber nicht plausibel sind

### 3.4 Zero-Glitch-Hold

Ein besonders störender Fehler sind einzelne kurze 0-W-Messungen mitten während eines stabilen Lade- oder Entladevorgangs.

Dafür gibt es den Zero-Glitch-Hold:

```text
ZERO_HOLD_W = 50 W
ZERO_CONFIRM_N = 2
```

Wenn der letzte gültige Wert betragsmäßig über 50 W lag und plötzlich exakt bzw. nahezu 0 W gelesen wird, wird dieser erste Nullwert nicht sofort übernommen.

Erst wenn **zwei aufeinanderfolgende 0-W-Messungen** auftreten, wird 0 W als echter Zustand akzeptiert.

Beispiel:

```text
700 W -> 698 W -> 0 W -> 702 W
```

Der einzelne Nullwert wird unterdrückt.

Dagegen:

```text
700 W -> 0 W -> 0 W
```

wird nach der Bestätigung als echtes Abschalten akzeptiert. Dadurch reagiert das System weiterhin schnell auf ein echtes Ende der Leistungsausgabe, ohne auf jeden einzelnen Null-Glitch zu springen.

### 3.5 Median-of-3

Nach den Plausibilitätsprüfungen werden die letzten drei akzeptierten Werte gespeichert. Aus diesen drei Werten wird der Median gebildet – also der mittlere Wert nach Größe, nicht der arithmetische Mittelwert.

Beispiel:

```text
698 W, 701 W, 1600 W
```

Median:

```text
701 W
```

Der einzelne Ausreißer mit 1600 W beeinflusst das Ergebnis praktisch nicht. Das ist für kurzfristige Einzelspitzen besser geeignet als ein einfacher Mittelwert, weil ein einzelner sehr großer Fehler den Mittelwert deutlich verschieben würde.

Das Ergebnis wird in `ac_stable_w` gespeichert und als **Venus AC Power W Stable** veröffentlicht.

### 3.6 Zwei veröffentlichte Leistungswerte

Die YAML stellt bewusst zwei verschiedene Leistungswerte bereit:

- `Venus AC Power W` – letzter akzeptierter plausibler Wert
- `Venus AC Power W Stable` – zusätzlich Median-of-3-geglätteter Wert

Für Anzeige, Heartbeat und Diagnose wird überwiegend die Stable-Variante verwendet.

### 3.7 ESPHome-Ausgabefilter

Zusätzlich besitzen die Template-Sensoren noch ESPHome-Filter:

```text
delta: 5 W
heartbeat: 5 s   (bei Stable Power)
```

`delta: 5` verhindert unnötige Zustandsupdates bei winzigen Änderungen. Der `heartbeat` sorgt gleichzeitig dafür, dass der stabile Leistungswert regelmäßig erneut veröffentlicht wird, auch wenn sich die Leistung nicht verändert.

Diese Filter dienen primär der sauberen Veröffentlichung nach Home Assistant. Die eigentliche Entglitchung findet bereits vorher in der Lambda-Logik statt.

## 4. SoC-Entglitchung

Auch der Ladezustand wird nicht blind übernommen.

Der Rohwert wird geprüft auf:

```text
0 % <= SoC <= 100 %
```

Anschließend wird der aktuelle Wert mit dem zuletzt akzeptierten SoC verglichen.

In der YAML gelten:

```text
MAX_JUMP = 8 Prozentpunkte
WINDOW_MS = 10 Minuten
```

Ein Sprung von mehr als 8 Prozentpunkten innerhalb von weniger als 10 Minuten wird zunächst verworfen.

Beispiel:

```text
76 % -> 75 % -> 20 % -> 75 %
```

Der einzelne Wert 20 % wird nicht übernommen. Das erklärt auch, warum `Venus SOC` gelegentlich für kurze Zeit noch den vorherigen Wert zeigen kann, obwohl im Debug-Log bereits ein anderer Rohwert auftaucht. Dieses Verhalten ist gewollt.

Der veröffentlichte SoC besitzt zusätzlich:

```text
delta: 1
heartbeat: 10 s
```

Dadurch wird Home Assistant nicht mit identischen oder minimalen Updates geflutet, bekommt aber trotzdem regelmäßig einen aktuellen Zustand.

## 5. Schutz der Graph-Historie

Für die interne kleine Leistungshistorie existiert noch ein zusätzlicher Clamp:

```text
-8000 W ... +8000 W
```

Dieser Clamp ist **kein eigentlicher Messwertfilter** für Home Assistant. Er schützt nur die interne Historie bzw. eine mögliche graphische Darstellung davor, dass ein extremer Ausreißer die Skalierung unbrauchbar macht.

## 6. Warum mehrere Filterstufen?

Die Kombination ist absichtlich redundant aufgebaut:

| Stufe | Schutz gegen |
|---|---|
| Hard-Reject | ungültige/defekte Extremwerte |
| Plausibilitätsgrenze | unrealistische, aber numerisch gültige Werte |
| Zero-Glitch-Hold | einzelne falsche 0-W-Messungen |
| Median-of-3 | kurze Einzelspitzen/Ausreißer |
| Delta-Filter | unnötige kleine Veröffentlichungsänderungen |
| Heartbeat | zu lange ausbleibende Sensor-Updates |
| SoC-Jumpfilter | unrealistische Ladezustandssprünge |
| Graph-Clamp | kaputte Darstellung durch Extremwerte |

Ziel ist nicht, die Messwerte künstlich träge zu machen. Echte Änderungen sollen weiterhin schnell sichtbar sein. Gefiltert werden hauptsächlich Muster, die typisch für Kommunikations- oder Einzelmessfehler sind.

## 7. Steuerregister

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

## 8. Master-Switch

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

## 9. NOT-AUS

Der Button `Venus NOT-AUS` setzt die Ausgabe sofort auf 0, deaktiviert RS485 Remote Control und setzt lokale Sollwerte zurück. Das ist kein Ersatz für die Schutzfunktionen des Speichers oder eine elektrische Not-Aus-Einrichtung, sondern ein Software-Stopp für dieses Projekt.

## 10. Request-Tracking und Underdelivery

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

## 11. Heartbeat

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

## 12. Modbus-Watchdog

Wenn für längere Zeit keine Modbus-Daten mehr empfangen werden:

1. nach >90 s wird der RS485-Master einmal aus- und wieder eingeschaltet
2. bleibt der Bus weiterhin tot, wird später der TTGO neu gestartet

Vor dem Recovery werden Sollwerte auf 0 gesetzt.

## 13. Display

Das TTGO-Display zeigt lokal:

- Modbus OK / Fehler
- WLAN-Qualität
- aktuellen Sollwert
- SoC
- aktuelle Lade- oder Entladeleistung
- farbliche Kennzeichnung der Richtung

So ist der Zustand des Speichers auch ohne Home Assistant direkt am Gerät sichtbar.

## 14. Tasten

Die beiden Tasten am T-Display verändern den lokalen Setpoint schrittweise und rufen anschließend dieselbe Modbus-Steuerlogik auf.

> **Hinweis:** Die Tastenlogik ist derzeit experimentell und am realen Referenzgerät noch nicht praktisch getestet.

## 15. Home Assistant / Node-RED

ESPHome veröffentlicht die relevanten Sensoren und Number-Entities nativ in Home Assistant. Eine Regelung kann deshalb entweder mit HA-Automationen oder Node-RED aufgebaut werden.

Die eigentliche Energie-Logik – z. B. Null-Einspeisung, Nachtprofil, Mindest-SoC oder PV-Überschuss – gehört bewusst nicht fest in dieses Repo. Das Projekt stellt die robuste lokale Schnittstelle zum Marstek bereit.

## 16. Firmware-Updates des Marstek

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
