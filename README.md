# Marstek Venus ESPHome HA RS485

Local control and monitoring of a **Marstek Venus** battery using a **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** and **Home Assistant**.

[🇩🇪 Deutsche README](README_DE.md)

The ESP32 communicates directly with the Venus over RS485. No cloud connection is required for the local control path. The TTGO display shows the most important information at the battery while Home Assistant exposes sensors, diagnostics and charge/discharge setpoints.

> **Status:** working project / reference implementation. Tested with a Marstek Venus E Gen3 and firmware V148. Marstek firmware changes can affect Modbus behaviour, so verify control after battery firmware updates.

## Finished installation

<p align="center"><img src="images/wall_setup.jpg" alt="Marstek Venus with TTGO RS485 controller" width="900"></p>

The compact TTGO controller is mounted next to the Marstek Venus and handles the local RS485 communication.

<p align="center"><img src="images/case.jpg" alt="3D printed TTGO enclosure" width="900"></p>

## Why this project?

- direct **RS485 / Modbus RTU** connection to the battery
- **ESPHome** firmware
- native **Home Assistant** entities
- **TTGO/LILYGO display** with SoC, power, Wi-Fi and Modbus status
- manual charge/discharge setpoints
- watchdog and Modbus recovery logic
- plausibility filters for SoC and AC power
- diagnostics for requested vs. measured battery power
- experimental physical-button setpoint control
- optional 3D-printed enclosure
- no cloud required for local control

## Hardware / parts list

| Part | Purpose | Notes | Link |
|---|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, Wi-Fi and display | ST7789, 135 × 240 px | [Amazon*](https://link.amazon/B0gITBNf5) |
| TTL ↔ RS485 transceiver module | Modbus/RS485 interface | Check 3.3 V logic compatibility | [Amazon*](https://link.amazon/B0evMwitV) |
| USB power supply + cable | TTGO power supply | Stable 5 V supply recommended | [Amazon*](https://link.amazon/B0cxu0tlI) |
| Standard Ethernet patch cable | Connection to Marstek | One end is cut off; blue, white/orange and orange are used in the shown build | any suitable cable |
| 3D-printed enclosure | Mechanical protection | Body + Cover STL included | [STL folder](enclosure/STL/) |

\* **Affiliate notice:** Links marked with `*` may be affiliate links. If you buy something through them, the project owner may receive a small commission. Your price does not change.

## Wiring

### ESP32 ↔ RS485 module

This mapping is defined by the current ESPHome configuration:

| TTGO / ESP32 | RS485 module | Function |
|---|---|---|
| GPIO27 | DI / TX | UART transmit |
| GPIO26 | RO / RX | UART receive |
| GPIO25 | DE + /RE | RS485 direction control |
| GND | GND | common ground |
| VCC | VCC | according to module specification |

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

### Marstek Venus ↔ controller: actual patch-cable wiring

The working reference build does **not use a special Marstek RS485 cable**. A normal Ethernet patch cable was cut at one end. The RJ45 plug remains connected to the Marstek Venus, while individual conductors from the cut end are wired into the controller.

Three conductors can be identified confidently from the actual build and photos:

- **blue**
- **white/orange**
- **orange**

At the small **3-pin connector inside the controller**, viewed as shown in the project photos, they are connected in this order:

```text
top      → blue
2nd pin  → white/orange
3rd pin  → orange
```

For a normally wired **T568B** patch cable these colours correspond to:

| Wire colour | T568B RJ45 pin | Role we can establish |
|---|---:|---|
| white/orange | 1 | one conductor of the RS485 differential pair |
| orange | 2 | the other conductor of the RS485 differential pair |
| blue | 4 | third conductor used in the actual build |

The working installation therefore confirms that **white/orange and orange form the RS485 data pair used by this build**. The original build did not record which of those two wires is labelled **A** and which is labelled **B** on the particular RS485 module, and the available photographs are not clear enough to make that assignment safely. This README therefore deliberately avoids inventing an A/B colour mapping.

The **blue conductor** is also connected in the working build. Its RJ45 pin position and the physical arrangement suggest a supply-related role, but its exact electrical assignment was not separately measured/documented during construction. Again, the documentation intentionally does not claim more than can be reconstructed from the working hardware.

> **Important:** Do not blindly connect three wires by colour. Patch cables may use a different wiring scheme. The RJ45 pin numbers are what matter. For a reproduction, identify the conductors from the RJ45 plug to the cut end with a multimeter/continuity tester.

> **RS485 A/B:** A and B naming is unfortunately not consistent between all RS485 adapters. If everything else is wired correctly but Modbus communication does not work, swapped differential data lines may be the reason. In that case swap only the two RS485 data conductors. Never experimentally swap a suspected supply conductor with a data conductor.

### What is known reliably on the TTGO side

In the prototype the patch-cable conductors first enter the internal wiring/RS485 board. The **TTGO-to-RS485-transceiver** signals, however, are unambiguous because they are defined in the YAML: TX = GPIO27, RX = GPIO26 and DE/RE = GPIO25. For a reproduction it is therefore better to follow these documented signals and the markings on your own RS485 module than to copy every internal prototype wire colour.

Reference parameters:

```text
Baud rate: 115200
Data bits: 8
Parity: NONE
Stop bits: 1
Modbus slave ID: 1
```

### Inside the controller

<p align="center"><img src="images/case_open.jpg" alt="Inside the TTGO controller" width="900"></p>

The TTGO T-Display and RS485 interface are housed together. The cables leave the enclosure through side cable glands.

## Display and the two buttons

<p align="center"><img src="images/display.jpg" alt="TTGO display in operation" width="900"></p>

The display shows the current setpoint (**SET**), state of charge (**SOC**), actual power and communication/Wi-Fi status.

> **Experimental / not yet hardware-tested:** The following button behaviour is implemented in the current YAML but has not yet been verified on the physical unit shown here. Test with small setpoints first.

The intended function of the two small TTGO buttons is:

- **Left button (GPIO0):** increases the setpoint by **100 W** per press. Positive setpoints mean **charging**.
- **Right button (GPIO35):** decreases the setpoint by **100 W** per press. Negative setpoints mean **discharging**.
- At **0 W** output is stopped.
- Firmware limits the local setpoint to **−2500 W … +2500 W**.
- Button commands are processed only while **Venus Master (RS485 Enable)** is enabled.

According to the current implementation, starting at 0 W and pressing left three times would select **+300 W charging**. Pressing right three times would return to 0 W; further right presses would move into discharge mode.

## ESPHome installation

1. Copy [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) into your ESPHome configuration directory.
2. Create/update `secrets.yaml` using [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml).
3. Check UART and RS485 pins against your hardware.
4. Validate the configuration in ESPHome.
5. Flash the TTGO.
6. Add the ESPHome device to Home Assistant.
7. Verify **Venus Modbus OK** before enabling automations.

## Home Assistant

<p align="center"><img src="images/HA%20Screenshot.jpg" alt="ESPHome device in Home Assistant" width="900"></p>

Entities include Venus SOC, AC Power, Stable Power, maximum charge/discharge power, temperatures, Modbus OK, charge/discharge setpoints, RS485 Master, emergency stop, OTA Quiet Mode, restart and WiFi RSSI.

## Why is there an OTA Quiet Mode?

The TTGO may be installed where Wi-Fi reception is weak. During OTA updates or troubleshooting we want to leave as much headroom as possible for the ESP32, Wi-Fi and communication tasks.

The display adds load in two ways: repeatedly drawing and refreshing the UI consumes CPU time and SPI activity, while the backlight consumes electrical power. **TTGO OTA Quiet Mode** is intended to reduce unnecessary display load. In the current configuration it switches off the display backlight, reducing display power consumption and leaving more supply headroom for the ESP32 and Wi-Fi. The regular display rendering currently continues in the background.

Quiet Mode does **not** disable the Marstek control logic. It is a service/troubleshooting mode for the TTGO itself.

## Control registers

| Register | Purpose | Values used |
|---:|---|---|
| `42000` | RS485 control enable | `21930` = enable, `21947` = disable |
| `42010` | Force mode | `0` standby, `1` charge, `2` discharge |
| `42020` | Forced charge power | 0…2500 W |
| `42021` | Forced discharge power | 0…2500 W |
| `42011` | Charge-to-SoC | % |
| `44002` | Max charge power | readback |
| `44003` | Max discharge power | readback |

### Firmware note

Marstek firmware can disable remote/RS485 control. The project therefore includes the RS485 control code in its heartbeat diagnostics. A healthy discharge heartbeat can look like:

```text
Heartbeat: ... req=-700 meas=797 rs485=21930 ctrl=2 dis_reg=700 ...
```

If the requested power and force mode look correct but the battery remains at 0 W, check `42000` / `rs485` first.

## What the firmware does

The YAML includes fast/slow Modbus polling, AC power plausibility and median filtering, SoC jump rejection, request/ack tracking, under-delivery diagnostics, Modbus freshness monitoring, recovery/watchdog logic, emergency stop, local TFT UI, experimental physical-button control and RS485 control-code readback in the heartbeat.

A section-by-section explanation is in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## 3D-printed enclosure

The enclosure consists of two printable parts:

- [`Body.stl`](enclosure/STL/Body.stl) – enclosure body
- [`Cover.stl`](enclosure/STL/Cover.stl) – front cover with display, button and ventilation openings

The files can be downloaded directly from the repository and opened in your slicer. The shown enclosure houses the TTGO, RS485 interface and wiring together.

## Credits / prior work

This project builds on work from the Marstek community. **ViperRNMC – marstek_venus_modbus** was an important reference for Marstek Venus Modbus registers, while **Superduper1969 – MarstekVenus-LilygoRS485** provided an ESPHome/LILYGO RS485 reference and inspiration.

## Support this project

If this project helped you and you want to support further development:

<a href="https://paypal.me/toor0001"><img src="assets/paypal-support-en.svg" alt="Buy me a coffee via PayPal" width="430"></a>

## Development note

This project was developed in a collaborative **vibe-coding workflow** using **ChatGPT** and **OpenAI Codex** for code generation, review, debugging and documentation.

The hardware design, integration decisions, practical testing and final responsibility for the project remain with the project author.

## Safety

This software can command battery charge and discharge power. Use it only if you understand your installation and battery limits. Keep the manufacturer's protection systems enabled and test changes at low power first.

This is an independent community project and is **not affiliated with or endorsed by Marstek, LILYGO, ESPHome or Home Assistant**.
