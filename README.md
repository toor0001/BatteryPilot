# Marstek Venus ESPHome HA RS485

Local control and monitoring of a **Marstek Venus** battery using a **LILYGO/TTGO T-Display**, **ESPHome**, **RS485/Modbus RTU** and **Home Assistant**.

[🇩🇪 Deutsche README](README_DE.md)

The ESP32 talks directly to the Venus over RS485. No cloud connection is required for the control path. The TTGO display shows the most important battery information locally, while Home Assistant exposes sensors, diagnostics and charge/discharge setpoints for automations or Node-RED.

> **Status:** working project / reference implementation. Tested with a Marstek Venus E Gen3 and firmware V148. Marstek firmware changes can affect Modbus behaviour, so verify control after battery firmware updates.

## Why this project?

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
- no cloud required for the local control path

## Hardware / parts list

| Part | Purpose | Notes | Link |
|---|---|---|---|
| LILYGO / TTGO T-Display ESP32 | Controller, Wi-Fi and local display | ST7789, 135 × 240 px | [Amazon*](https://link.amazon/B0gITBNf5) |
| TTL ↔ RS485 transceiver module | Physical Modbus/RS485 interface | Check 3.3 V logic compatibility | [Amazon*](https://link.amazon/B0evMwitV) |
| USB power supply + cable | TTGO power supply | Stable 5 V supply recommended | [Amazon*](https://link.amazon/B0cxu0tlI) |
| RS485 cable | Connection to Venus | Twisted A/B pair recommended | link follows |
| Suitable connector for Marstek RS485 port | Battery connection | Verify pinout on your device | link follows |
| 3D-printed enclosure | Mechanical protection | STL files in `enclosure/STL/` | included |

\* **Affiliate notice:** Links marked with `*` may be affiliate links. If you buy something through them, the project owner may receive a small commission. Your price does not change.

## Wiring

### ESP32 ↔ RS485 module

| TTGO / ESP32 | RS485 module | Function |
|---|---|---|
| GPIO27 | DI / TX | UART transmit |
| GPIO26 | RO / RX | UART receive |
| GPIO25 | DE + /RE | RS485 direction control |
| GND | GND | common ground |
| VCC | VCC | according to your RS485 module specification |

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

At least the differential **A** and **B** lines are connected. Depending on the installation/module, a common GND can also be useful or required.

> A/B naming is unfortunately not consistent between all RS485 module vendors. If there is no communication at all, swapped A/B lines are one of the first things to check.

Reference bus settings:

```text
Baud rate: 115200
Data bits: 8
Parity: NONE
Stop bits: 1
Modbus slave ID: 1
```

### TTGO T-Display pins

| Function | GPIO |
|---|---:|
| SPI CLK | 18 |
| SPI MOSI | 19 |
| TFT CS | 5 |
| TFT DC | 16 |
| TFT RESET | 23 |
| Backlight PWM | 4 |
| left button | 0 |
| right button | 35 |

## ESPHome installation

1. Copy [`esphome/marstek-venus-ttgo.yaml`](esphome/marstek-venus-ttgo.yaml) into your ESPHome configuration directory.
2. Create/update `secrets.yaml` using [`esphome/secrets.example.yaml`](esphome/secrets.example.yaml).
3. Check UART and RS485 pins against your hardware.
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
- TTGO OTA Quiet Mode
- TTGO Restart
- TTGO WiFi RSSI

## Why is there an OTA Quiet Mode?

The TTGO may be installed in a location with weak Wi-Fi reception. During OTA updates or troubleshooting we want to leave as much headroom as possible for the ESP32, Wi-Fi and communication tasks.

The display adds load in two ways: repeatedly drawing and refreshing the UI consumes CPU time and SPI activity, while the backlight also consumes electrical power. **TTGO OTA Quiet Mode** is intended to minimise unnecessary display load in these situations. In the current configuration it switches off the display backlight, reducing display power consumption and leaving more supply headroom for the ESP32 and Wi-Fi.

It is particularly useful when:

- the TTGO has weak Wi-Fi reception,
- an OTA update is unstable,
- you want to minimise unnecessary display load while diagnosing connectivity.

The mode does **not** disable the Marstek control logic. It is a service/troubleshooting mode for the TTGO itself.

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

Marstek firmware can disable remote/RS485 control when the operating mode is changed from the app or via the user-work-mode register. The project therefore keeps the RS485 control state visible in diagnostics.

A healthy discharge heartbeat can look like:

```text
Heartbeat: ... req=-700 meas=797 rs485=21930 ctrl=2 dis_reg=700 ...
```

If the requested power is present but the battery stays at 0 W, check `42000` / `rs485` first.

## What the firmware does

The YAML includes:

- separate fast and slow Modbus polling controllers
- AC-power sanity checking and median-of-3 filtering
- SoC jump rejection
- request/ack tracking
- under-delivery diagnostics
- Modbus freshness monitoring
- one-shot RS485 recovery if data becomes stale
- automatic TTGO restart if recovery fails
- emergency stop logic
- local TFT status screen and button control
- RS485 control-code readback in the heartbeat

A section-by-section explanation is in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).

## Before first use

1. Check RS485 module supply voltage and logic levels.
2. Check GND and A/B wiring.
3. Start without automatic charge/discharge automation.
4. Confirm `Venus Modbus OK`.
5. Check SoC and power values for plausibility.
6. Test small setpoints first, e.g. 100–300 W.
7. Only then enable unattended automation.

## 3D-printed enclosure

STL files are located under [`enclosure/STL/`](enclosure/STL/).

## Credits / prior work

This project stands on work from the Marstek community. In particular:

- **ViperRNMC – marstek_venus_modbus** – important reference for Marstek Venus Modbus register definitions and Home Assistant integration behaviour.
- **Superduper1969 – MarstekVenus-LilygoRS485** – ESPHome/LILYGO RS485 reference implementation and inspiration for a direct ESP-based Marstek interface.

## Support this project

If this project helped you and you want to support further development:

<a href="https://paypal.me/toor0001"><img src="assets/paypal-support-en.svg" alt="Buy me a coffee via PayPal" width="430"></a>

## Safety

This software can command battery charge and discharge power. Use it only if you understand your installation and the battery limits. Keep the manufacturer's protection systems enabled and test changes at low power first.

This is an independent community project and is **not affiliated with or endorsed by Marstek, LILYGO, ESPHome or Home Assistant**.
