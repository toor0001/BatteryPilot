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

The main reason for this project is not simply to display Marstek data in Home Assistant. It is to make the **Marstek Venus externally controllable as one component of a larger home energy-management system**.

In the original installation, PV surplus is distributed between several flexible loads according to a custom priority scheme:

1. **Electric vehicle first:** as much available PV surplus as possible should be used to charge the EV.
2. **Hot water second:** if the car is not charging, is full, or cannot absorb the available surplus, the remaining energy is sent to an electric immersion heater for domestic hot water. A Node-RED flow controls the heater in several power stages between **0 and 3000 W**.
3. **Battery third:** only when the EV does not need the energy and the water is already warm enough should the remaining surplus charge the Marstek Venus.

The Marstek battery is therefore used primarily as a **time-shifting storage device**: it stores otherwise unused PV energy and is intended mainly to cover the house load later, especially during the night.

The reverse direction is just as important. The Marstek must **not discharge in order to feed the EV charger or the immersion heater**. From the battery's point of view, those consumers could otherwise simply look like additional house load. That would defeat the intended priority scheme by moving previously stored energy into loads that should only consume current PV surplus.

For this reason, the built-in Marstek control logic alone was not sufficient for this installation. The battery needed to become a controllable participant in the home's central energy-management logic. This ESPHome project provides that missing interface: via **RS485 / Modbus RTU**, Home Assistant can read the relevant operating data and explicitly command Marstek charge and discharge power.

### What this repository does — and what it deliberately does not do

This repository provides the **general-purpose Marstek integration and control layer**:

- direct **RS485 / Modbus RTU** connection to the battery
- **ESPHome** firmware
- native **Home Assistant** entities
- externally adjustable charge/discharge setpoints
- **TTGO/LILYGO display** with SoC, power, Wi-Fi and Modbus status
- watchdog and Modbus recovery logic
- plausibility filters for SoC and AC power
- diagnostics for requested vs. measured battery power
- experimental physical-button setpoint control
- optional 3D-printed enclosure
- no cloud required for local control

The actual **Node-RED energy-management logic is intentionally not part of this repository**. Such logic depends heavily on the individual PV system, wallbox, EV, hot-water system, household loads and desired priorities. This project is therefore not intended to prescribe how a house should distribute its energy. Its purpose is to make the Marstek Venus **generically readable and controllable from Home Assistant**, so that it can be integrated into any suitable higher-level automation — Node-RED, Home Assistant automations or another controller.

## Hardware / parts list

| Part | Purpose | Notes | Link |
|---|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, Wi-Fi and display | ST7789, 135 × 240 px | [Amazon*](https://link.amazon/B0gITBNf5) |
| TTL ↔ RS485 transceiver module | Modbus/RS485 interface | Check 3.3 V logic compatibility | [Amazon*](https://link.amazon/B0evMwitV) |
| 5 V power supply | Permanent controller supply | In the reference build +5 V and GND are fed to the green perfboard/carrier board; Micro-USB is not the normal installed power input | [Amazon*](https://link.amazon/B0cxu0tlI) |
| Standard Ethernet patch cable | Connection to Marstek | One end is cut off; blue, white/orange and orange are used in the shown build | any suitable cable |
| 3D-printed enclosure | Mechanical protection | Body + Cover STL included | [STL folder](enclosure/STL/) |

\* **Affiliate notice:** Links marked with `*` may be affiliate links. If you buy something through them, the project owner may receive a small commission. Your price does not change.

## Wiring

### Complete overview

The finished controller has two separate paths: **power** and **RS485 communication**.

```text
230 V AC
   │
   ▼
5 V power supply
   │  +5 V / GND
   ▼
green perfboard / carrier board
   │
   ├──► TTGO T-Display
   └──► RS485 electronics / common supply

TTGO T-Display
   │ GPIO27 / GPIO26 / GPIO25 / GND
   ▼
RS485 transceiver
   │ A / B / reference conductor
   ▼
cut Ethernet patch cable
   │
   ▼
Marstek Venus RS485 port
```

### Power: supply → carrier board → TTGO

In the **permanently installed reference build**, the TTGO is **not powered through its Micro-USB connector**. The external power supply provides 5 V. **+5 V and GND are connected to the green perfboard/carrier board** on which the TTGO is soldered. The controller is powered through this board, which also carries the internal wiring to the RS485 interface.

A USB cable visible on some project/test photos was used only for **testing/flashing** and is not the normal power connection of the installed controller.

> **Important:** Use a regulated 5 V supply and verify voltage and polarity before connection. Do not simultaneously connect an unknown external 5 V feed and USB unless you have verified that your particular hardware safely supports it.

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

Three conductors can be identified confidently from the actual build and photos: **blue**, **white/orange**, and **orange**.

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

The **blue conductor** is also connected in the working build. Its exact electrical assignment was not separately measured/documented during construction, so the documentation deliberately does not claim whether it is GND or another function.

> **Important:** Do not blindly connect three wires by colour. Patch cables may use a different wiring scheme. The RJ45 pin numbers are what matter. For a reproduction, identify the conductors from the RJ45 plug to the cut end with a multimeter/continuity tester.

> **RS485 A/B:** A and B naming is unfortunately not consistent between all RS485 adapters. If everything else is wired correctly but Modbus communication does not work, swapped differential data lines may be the reason. In that case swap only the two RS485 data conductors. Never experimentally swap the third conductor with a data conductor.

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

The TTGO T-Display, green carrier board and RS485 interface are housed together. The cables leave the enclosure through side cable glands.

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
