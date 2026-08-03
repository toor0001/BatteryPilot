# Marstek Venus ESPHome HA RS485

Local control and monitoring of a **Marstek Venus** battery using a **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** and **Home Assistant**.

The ESP32 talks directly to the Venus over RS485. No cloud connection is required for the control path. The TTGO display shows the most important battery information locally, while Home Assistant exposes sensors, diagnostics and charge/discharge setpoints for automations or Node-RED.

> **Status:** working project / reference implementation. Tested by the project author with a Marstek Venus E Gen3 and firmware V148. Marstek firmware changes can affect Modbus behaviour, so read the firmware notes below before updating a battery.

## Why this project?

There are already excellent Marstek Modbus projects, but this build combines the pieces into one small local device:

- direct **RS485 / Modbus RTU** connection to the battery
- **ESPHome** firmware
- native **Home Assistant** entities
- **TTGO/LILYGO display** with SoC, power, Wi-Fi and Modbus status
- manual charge/discharge setpoints
- watchdog and Modbus recovery logic
- plausibility filters for SoC and AC power
- diagnostics for requested vs. measured battery power
- physical buttons on the TTGO for local setpoint control
- optional 3D-printed enclosure

## Repository layout

```text
.
├── README.md
├── esphome/
│   ├── marstek-venus-ttgo.yaml
│   └── secrets.example.yaml
├── docs/
│   ├── CODE_EXPLAINED.md
│   └── HARDWARE.md
└── enclosure/
    └── STL/
        └── README.md
```

## Hardware

The reference build uses:

- **LILYGO / TTGO T-Display ESP32** with 135 × 240 ST7789 display
- **3.3 V compatible TTL ↔ RS485 transceiver/module**
- Marstek Venus battery with RS485 interface
- suitable RS485 cable/connector for the Venus
- USB power supply/cable for the TTGO
- optional 3D-printed enclosure

See [`docs/HARDWARE.md`](docs/HARDWARE.md) for the wiring and important electrical notes.

## Wiring used by the ESPHome configuration

| TTGO / ESP32 | RS485 module | Function |
|---|---|---|
| GPIO27 | DI / TX | UART transmit |
| GPIO26 | RO / RX | UART receive |
| GPIO25 | DE + /RE | RS485 direction control |
| GND | GND | common ground |
| supply | VCC | according to your RS485 module specification |

RS485 **A/B** then connects to the Marstek RS485 bus. If communication does not work, check your module documentation first; A/B naming is unfortunately not consistent between all vendors.

## ESPHome installation

1. Copy [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) into your ESPHome configuration directory.
2. Create/update `secrets.yaml` using [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml).
3. Check the UART and RS485 pins against your hardware.
4. Validate the configuration in ESPHome.
5. Flash the TTGO.
6. Add the ESPHome device to Home Assistant.
7. Verify **Venus Modbus OK** before enabling automations.

## Home Assistant entities

The configuration exposes, among others:

- Venus SOC
- Venus AC Power
- Venus AC Power W Stable
- Venus Max Charge Power
- Venus Max Discharge Power
- Venus Charge To SoC
- cell/internal temperature sensors
- Venus Modbus OK
- Venus Set Charge Power W
- Venus Set Discharge Power W
- Venus Master (RS485 Enable)
- Venus NOT-AUS
- TTGO Restart
- TTGO WiFi RSSI

The low-level control registers themselves are intentionally kept `internal: true` where possible.

## Control registers

The current Gen3 control path uses these holding registers:

| Register | Purpose | Values used |
|---:|---|---|
| `42000` | RS485 control enable | `21930` = enable, `21947` = disable |
| `42010` | Force mode | `0` standby, `1` charge, `2` discharge |
| `42020` | Forced charge power | 0…2500 W |
| `42021` | Forced discharge power | 0…2500 W |
| `42011` | Charge-to-SoC | % |
| `44002` | Max charge power | readback |
| `44003` | Max discharge power | readback |

### Important firmware behaviour

Marstek firmware can disable remote/RS485 control when the operating mode is changed from the app or via the user-work-mode register. The project therefore keeps the RS485 control state visible in diagnostics. A healthy discharge heartbeat can look like:

```text
Heartbeat: ... rs485=21930 ctrl=2 dis_reg=700 ...
```

If the requested power is present in `42021` but the battery stays at 0 W, check `42000` first.

Do **not** assume that a firmware update preserves all Modbus behaviour. After a battery firmware update, verify readback and control manually before re-enabling unattended automations.

## What the firmware does

The YAML is deliberately more defensive than a minimal Modbus example. It includes:

- separate fast and slow Modbus polling controllers
- AC-power sanity checking and median-of-3 filtering
- SoC jump rejection
- request/ack tracking
- under-delivery diagnostics
- a Modbus freshness sensor
- one-shot RS485 recovery if data becomes stale
- automatic TTGO restart if recovery fails
- emergency stop logic
- local TFT status screen and button control

A section-by-section explanation is in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## 3D-printed enclosure

A directory for printable enclosure files is already prepared under [`enclosure/STL/`](enclosure/STL/). The final STL files can be added there without changing the documentation structure.

## Credits / prior work

This project stands on work from the Marstek community. In particular:

- **ViperRNMC – marstek_venus_modbus**  
  https://github.com/ViperRNMC/marstek_venus_modbus  
  Important reference for current Marstek Venus Modbus register definitions and Home Assistant integration behaviour.

- **Superduper1969 – MarstekVenus-LilygoRS485**  
  https://github.com/Superduper1969/MarstekVenus-LilygoRS485  
  Excellent ESPHome/LILYGO RS485 reference implementation and inspiration for a direct ESP-based Marstek interface.

Please also follow the upstream credits in those projects; the Marstek register knowledge is community work built over time.

## Safety

This software can command battery charge and discharge power. Use it only if you understand your installation and the battery limits. Keep the manufacturer's protection systems enabled and test changes at low power first.

This is an independent community project and is **not affiliated with or endorsed by Marstek, LILYGO, ESPHome or Home Assistant**.
