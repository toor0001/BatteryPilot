# Marstek Venus Modbus Implementation on ESP32 and ESPHome

Local control and monitoring of a **Marstek Venus** battery using a **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** and **Home Assistant**.

<a href="README_DE.md"><img src="https://flagcdn.com/24x18/de.png" width="24" height="18" alt="Deutsch"> Deutsche README</a>

[![ESPHome](https://img.shields.io/badge/ESPHome-ESP32-blue)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-compatible-blue)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

This project allows the Marstek Venus to be charged and discharged directly and conveniently from Home Assistant. No cloud connection and no Marstek **Local API** are required. The TTGO receives commands from Home Assistant via Wi-Fi and forwards them to the battery via Modbus. It exposes sensors, diagnostics and charge/discharge setpoints in Home Assistant, while the TTGO display shows the most important information locally at the device.

> **Project status: completed / reference implementation.** Tested with a Marstek Venus E Gen3 and firmware V148. This project is not under active development. Updates will only be made when necessary, for example if Marstek firmware changes affect RS485/Modbus communication. After battery firmware updates, always verify that control still works as expected.

## Finished installation

<p align="center"><img src="images/wall_setup.jpg" alt="Marstek Venus with TTGO RS485 controller" width="900"></p>

The compact TTGO controller is mounted next to the Marstek Venus and handles the local RS485 communication.

<p align="center"><img src="images/case.jpg" alt="3D printed TTGO enclosure" width="900"></p>

## Why this project?

The main purpose of this project is not simply to display Marstek data in Home Assistant. It is to make the **Marstek Venus a controllable part of a higher-level home energy-management system**.

In my house, PV surplus is distributed according to a custom priority scheme:

1. **EV first:** as much available PV surplus as possible should go to the electric vehicle.
2. **Hot water second:** if the EV is not charging or is full, a Node-RED flow controls an immersion heater in several stages between **0 and 3000 W**.
3. **Marstek last:** only when neither the EV nor hot water needs the surplus is the Marstek charged.

In addition, I use various Zigbee devices, for example to request a hot-water **boost** while showering when the available hot water is getting low. The Marstek itself is mainly used as a **time-shifting energy store**, keeping otherwise unused PV energy for later household consumption, especially at night. In the other direction, it must **not discharge in order to feed the EV or immersion heater**.

I have been controlling this overall logic with a proven Node-RED flow for years. Because of this more complex prioritisation, I cannot use the Marstek's own automatic self-consumption optimisation for my installation.

This repository only provides the general-purpose **ESPHome / RS485 / Home Assistant interface**, which can then be used by automations or simply for visualisation in Home Assistant. The actual Node-RED energy-management logic is deliberately **not part of this repository**, because it depends strongly on the individual PV system, wallbox, EV, hot-water system and personal priorities.

## Hardware / parts list

| Part | Purpose | Notes | Link |
|---|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, Wi-Fi and local display | ST7789, 135 × 240 px | [Amazon*](https://link.amazon/B0gITBNf5) |
| TTL ↔ RS485 transceiver module | Modbus/RS485 interface | module used here exposes `3-5V`, `RX-I`, `TX-O`, `RTS`, `GND` and `A/B/G` | [Amazon*](https://link.amazon/B0evMwitV) |
| 5 V power supply | Permanent controller supply | +5 V and GND feed the green carrier/perfboard; Micro-USB is not the normal installed supply | [Amazon*](https://link.amazon/B047QzbuV) |

\* **Affiliate notice:** Links marked with `*` may be affiliate links. If you buy through them, the project owner may receive a small commission. Your price does not change.

## Wiring

### Complete overview

The Marstek does **not power the controller** in this build. Both TTGO and RS485 transceiver are powered from the **external 5 V supply**. The Marstek side carries only the three RS485 conductors **A, B and G**.

```text
230 V AC
   │
   ▼
5 V power supply
   │  +5 V / GND
   ▼
green carrier / perfboard
   ├──────────────► TTGO T-Display
   └──────────────► RS485 module: 3-5V + GND

TTGO T-Display                  RS485 module
GPIO27 (TX) ──────────────────► RX-I
GPIO26 (RX) ◄────────────────── TX-O
GPIO25      ──────────────────► RTS
GND         ─────────────────── GND

RS485 module                    T568B patch cable      Marstek RJ45
A ────────────────────────────► white/orange ────────► pin 1
B ────────────────────────────► orange ─────────────► pin 2
G ────────────────────────────► blue ───────────────► pin 4
```

### Complete connection table

| From | To | Function |
|---|---|---|
| external supply +5 V | green carrier board | power |
| external supply GND | green carrier board | ground |
| carrier board +5 V | TTGO 5 V input | TTGO power |
| carrier board GND | TTGO GND | TTGO ground |
| carrier board +5 V | RS485 `3-5V` | transceiver power |
| carrier board GND | RS485 `GND` | transceiver ground |
| TTGO GPIO27 (TX) | RS485 `RX-I` | UART TTGO → transceiver |
| TTGO GPIO26 (RX) | RS485 `TX-O` | UART transceiver → TTGO |
| TTGO GPIO25 | RS485 `RTS` | transmit/receive direction |
| RS485 `A` | T568B white/orange | Marstek RS485 A, RJ45 pin 1 |
| RS485 `B` | T568B orange | Marstek RS485 B, RJ45 pin 2 |
| RS485 `G` | T568B blue | RS485 reference, RJ45 pin 4 |

### Power supply

The external supply feeds 5 V to the green carrier/perfboard. From there, both the TTGO and RS485 module are powered.

> **Important:** Verify voltage and polarity before connecting. Do not connect an external 5 V feed and USB at the same time unless you have verified that your specific hardware safely supports it.

### ESPHome pins

The TTGO-side pins are also fixed by the current YAML:

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

Reference parameters:

```text
Baud rate: 115200
Data bits: 8
Parity: NONE
Stop bits: 1
Modbus slave ID: 1
```

### Patch cable / RJ45

The reference build uses a normal **T568B Ethernet patch cable** cut at one end. The RJ45 plug remains connected to the Marstek; only three conductors are used at the open end:

| RJ45 pin | T568B colour | RS485 module |
|---:|---|---|
| 1 | white/orange | A |
| 2 | orange | B |
| 4 | blue | G |

At the small 3-pin connector inside the controller, **in the orientation shown in the project photos**, the wires are top-to-bottom **blue → white/orange → orange**.

> **Important:** Do not blindly copy wire colours. With a different or differently wired patch cable, identify RJ45 pins 1, 2 and 4 using a continuity tester.

### Inside the controller

<p align="center"><img src="images/case_open.jpg" alt="Inside the controller" width="900"></p>

The TTGO, carrier board, power supply and RS485 interface are housed together.

## Display and the two buttons

<p align="center"><img src="images/display.jpg" alt="TTGO display in operation" width="900"></p>

The display shows the setpoint (**SET**), state of charge (**SOC**), current power and communication/Wi-Fi status.

> **Experimental / not yet hardware-tested:** The button behaviour is implemented in the YAML but has not yet been verified on the physical unit shown here.

- **Upper button (GPIO0):** +100 W per press; positive values mean charging.
- **Lower button (GPIO35):** −100 W per press; negative values mean discharging.
- At **0 W** output is stopped.
- Range: **−2500 W to +2500 W**.
- Only active while **Venus Master (RS485 Enable)** is enabled.

## ESPHome installation

1. Create a new device in ESPHome and select **Espressif ESP32 Dev Module**.
2. Copy [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) into ESPHome.
3. Create `secrets.yaml` using [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml).
4. Validate the YAML and flash the TTGO, for example by connecting it via USB directly to a PC or the Home Assistant host.
5. Home Assistant should discover the device under **Settings → Devices & services**; add it there.

## Marstek app settings / operating mode

This project controls the Venus through **physical RS485 / Modbus RTU**, not through Marstek's network-based Local API.

For the **tested V148 reference setup**:

- **Local API / Open API does not need to be enabled.** It is a separate interface and is not required by this project.
- Set the Marstek operating mode to **Manual** before handing charge/discharge control to the external RS485 automation.
- Avoid changing operating modes from the Marstek app while RS485 control is active. Firmware behaviour can reset or disable external control when modes are changed.
- Charge and discharge decisions are then made by your Home Assistant / Node-RED logic through the entities exposed by this project.

### What about a Marstek smart meter?

This setup deliberately replaces the Marstek's own automatic control loop with an external one. A Marstek smart meter may still be physically installed or used as a measurement source, but **it should not be expected to control battery charge/discharge while the Venus is being force-controlled through this RS485 setup**.

In Manual/force-control operation, the external automation must therefore make the decisions itself. That also means it must prevent unwanted behaviour such as the battery discharging into an EV charger, immersion heater or another flexible load.

## Home Assistant

<p align="center"><img src="images/HA%20Screenshot.jpg" alt="ESPHome device in Home Assistant" width="900"></p>

Entities include SoC, AC power, stable/filtered power, temperatures, maximum charge/discharge power, Modbus status, charge/discharge setpoints, RS485 Master, emergency stop, OTA Quiet Mode, restart and WiFi RSSI.

## Why is there an OTA Quiet Mode?

With weak Wi-Fi reception, the ESP32 should retain as much headroom as possible for radio and communication tasks. Display rendering creates CPU/SPI load, while the backlight consumes additional electrical power. In the current configuration Quiet Mode switches off the backlight, reducing electrical load; display rendering currently continues in the background. Marstek control remains active.

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

Marstek firmware can disable remote/RS485 control again. The RS485 control code is therefore included in the heartbeat:

```text
Heartbeat: ... req=-700 meas=797 rs485=21930 ctrl=2 dis_reg=700 ...
```

If setpoint and force mode look correct but the battery remains at 0 W, check `42000` / `rs485` first.

## What the firmware does

The YAML includes fast/slow Modbus polling, multi-stage plausibility and anti-glitch filtering, median filtering, SoC jump rejection, request/ack tracking, under-delivery diagnostics, Modbus freshness monitoring, recovery/watchdog logic, emergency stop, TFT UI and RS485 control-code readback in the heartbeat.

A detailed explanation is in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## 3D-printed enclosure

- [`Body.stl`](enclosure/STL/Body.stl) – enclosure body
- [`Cover.stl`](enclosure/STL/Cover.stl) – front cover with display, button and ventilation openings

## Credits / prior work

This project builds on work from the Marstek community. **ViperRNMC – marstek_venus_modbus** was an important reference for Marstek Venus Modbus registers, while **Superduper1969 – MarstekVenus-LilygoRS485** provided an ESPHome/LILYGO RS485 reference and inspiration.

A review of the current YAML did **not identify substantial verbatim copying of application logic** from either project. Shared elements are mainly ESPHome/Modbus structures, public register addresses/command values and normal RS485 concepts. The custom filtering, watchdog/recovery, TFT UI, request/ack and diagnostic logic in this repository are separate implementations.

See [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) for attribution and licensing notes.

## License

This repository is released under the **MIT License**. See [`LICENSE`](LICENSE).

## Say thanks

If this project helped you or you were able to use it successfully for your own setup, I appreciate a small thank-you. You can use the **Sponsor** button at the top of the repository or buy me a coffee directly via PayPal.

<a href="https://paypal.me/toor0001"><img src="assets/paypal-support-en.svg" alt="Buy me a coffee via PayPal" width="430"></a>

## Development note

This project was developed in a collaborative **vibe-coding workflow** using **ChatGPT** and **OpenAI Codex** for code generation, review, debugging and documentation.

The hardware design, integration decisions, practical testing and final responsibility for the project remain with the project author.

## Safety

This software can command battery charge and discharge power. Use it only if you understand your installation and battery limits. Keep the manufacturer's protection systems enabled and test changes at low power first.

This is an independent community project and is **not affiliated with or endorsed by Marstek, LILYGO, ESPHome or Home Assistant**.
