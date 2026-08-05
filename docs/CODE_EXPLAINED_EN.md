# Code Explained

[Deutsch](CODE_EXPLAINED.md) · English

The ESPHome configuration is more than a simple list of registers. It handles communication, filtering, control, diagnostics, watchdog functions, and the local display on the TTGO.

## 1. Boot behaviour

During startup, the RS485 master is initially disabled. Remote control is enabled only after a short delay and once a Wi-Fi connection has been established successfully. This prevents old setpoints from being written to the battery immediately during boot.

Important states:

- `master_enabled` – local enable state for RS485 control
- `last_rx_ms` – timestamp of the last valid Modbus response
- `boot_ms` – startup time used for watchdog grace periods
- `wd_recover_attempted` – prevents endless recovery loops

## 2. Two Modbus controllers

The configuration uses two polling speeds:

```text
venus_fast  -> fast-changing values such as power and SoC
venus_slow  -> limits, temperatures, and configuration values
```

This keeps power and SoC responsive without continuously loading the RS485 bus with all slower registers.

## 3. Multi-stage measurement processing / anti-glitch pipeline

Raw values from the Venus are not passed directly to Home Assistant or the display without validation. Serial communication can occasionally produce individual incorrect or briefly implausible readings. AC power therefore passes through several protection stages. In practice, this works very reliably.

The simplified data path looks like this:

```text
Modbus RAW
   ↓
Hard reject
   ↓
Plausibility check
   ↓
Zero-glitch confirmation
   ↓
Median-of-3
   ↓
Stable Power
   ↓
Home Assistant / Display / Diagnostics
```

The important point is that these filters serve different purposes. A median filter alone, for example, would not reliably protect against a genuine communication error that produces a very large numeric value. Conversely, a simple range check would not reliably suppress brief 0 W glitches.

### 3.1 AC power raw value

Raw AC power is read from register `30006` and is first processed internally as `Venus AC Power raw (fast)`. Every valid incoming reading also updates `last_rx_ms`. The same data stream therefore acts as a heartbeat for Modbus communication.

### 3.2 Hard reject of invalid values

Obviously broken values are rejected first:

```text
not finite / NaN / Inf  -> reject
|value| > 20000 W       -> reject
```

The 20 kW limit is a hard safety boundary against completely incorrect register values or decoding outliers.

### 3.3 Plausibility limit

A narrower limit is then applied for the realistically expected operating range:

```text
|value| > 6500 W -> reject
```

The reference battery operates well below this power level. Unrealistic values therefore never reach the subsequent processing stages. The two limits are intentionally separate:

- `ABS_MAX = 20000 W` protects against obviously defective data
- `PLAUS_MAX = 6500 W` protects against values that are numerically representable but not plausible for this installation

### 3.4 Zero-glitch hold

A particularly disruptive error is an isolated brief 0 W reading in the middle of a stable charging or discharging period.

The zero-glitch hold handles this case:

```text
ZERO_HOLD_W = 50 W
ZERO_CONFIRM_N = 2
```

If the magnitude of the last valid value was above 50 W and the next reading suddenly becomes exactly or nearly 0 W, that first zero value is not accepted immediately.

Only when **two consecutive 0 W readings** occur is 0 W accepted as the real state.

Example:

```text
700 W -> 698 W -> 0 W -> 702 W
```

The isolated zero value is suppressed.

By contrast:

```text
700 W -> 0 W -> 0 W
```

is accepted as a genuine shutdown after confirmation. The system therefore still reacts quickly when power output actually ends, without reacting to every individual zero glitch.

### 3.5 Median-of-3

After the plausibility checks, the last three accepted values are stored. Their median is calculated, meaning the middle value by magnitude rather than the arithmetic mean.

Example:

```text
698 W, 701 W, 1600 W
```

Median:

```text
701 W
```

The isolated 1600 W outlier has virtually no effect on the result. This is better suited to short individual spikes than a simple average because one very large error would shift the average significantly.

The result is stored in `ac_stable_w` and published as **Venus AC Power W Stable**.

### 3.6 Two published power values

The YAML intentionally provides two different power values:

- `Venus AC Power W` – the most recent accepted plausible value
- `Venus AC Power W Stable` – additionally smoothed using median-of-3

The Stable variant is used primarily for the display, heartbeat, and diagnostics.

### 3.7 ESPHome output filters

The template sensors also use ESPHome filters:

```text
delta: 5 W
heartbeat: 5 s   (for Stable Power)
```

`delta: 5` prevents unnecessary state updates caused by tiny changes. At the same time, `heartbeat` ensures that the stable power value is republished regularly even when the power does not change.

These filters primarily keep publishing to Home Assistant clean and efficient. The actual de-glitching already takes place earlier in the lambda logic.

## 4. SoC de-glitching

The state of charge is also validated rather than accepted blindly.

The raw value is checked against:

```text
0 % <= SoC <= 100 %
```

The current value is then compared with the last accepted SoC.

The YAML uses:

```text
MAX_JUMP = 8 percentage points
WINDOW_MS = 10 minutes
```

A jump of more than 8 percentage points within less than 10 minutes is initially rejected.

Example:

```text
76 % -> 75 % -> 20 % -> 75 %
```

The isolated 20 % value is not accepted. This also explains why `Venus SOC` may occasionally continue to show the previous value briefly even though a different raw value already appears in the debug log. This behaviour is intentional.

The published SoC additionally uses:

```text
delta: 1
heartbeat: 10 s
```

This prevents Home Assistant from being flooded with identical or minimal updates while still receiving a current state regularly.

## 5. Protecting graph history

The internal short power history has an additional clamp:

```text
-8000 W ... +8000 W
```

This clamp is **not an actual measurement filter** for Home Assistant. It only protects the internal history or a possible graph display from having its scale made unusable by an extreme outlier.

## 6. Why use several filter stages?

The combination is deliberately redundant:

| Stage | Protects against |
|---|---|
| Hard reject | invalid or defective extreme values |
| Plausibility limit | unrealistic but numerically valid values |
| Zero-glitch hold | isolated false 0 W readings |
| Median-of-3 | brief individual spikes and outliers |
| Delta filter | unnecessary small publication changes |
| Heartbeat | sensor updates being absent for too long |
| SoC jump filter | unrealistic state-of-charge jumps |
| Graph clamp | broken graph scaling caused by extreme values |

The aim is not to make measurements artificially sluggish. Genuine changes should remain visible quickly. The filters mainly target patterns typical of communication errors or isolated measurement faults.

## 7. Control registers

The Gen3 control uses the registers known from the community:

| Register | Meaning |
|---:|---|
| 42000 | RS485 Remote Control |
| 42010 | Force Mode |
| 42020 | Charge power |
| 42021 | Discharge power |

### RS485 Remote Control

```text
21930 = enable Remote Control
21947 = disable Remote Control
```

### Force Mode

```text
0 = Standby
1 = Charge
2 = Discharge
```

The Home Assistant entities `Venus Set Charge Power W` and `Venus Set Discharge Power W` write these registers through ESPHome scripts.

## 8. Master switch

`Venus Master (RS485 Enable)` is the central enable switch.

When switched on:

```text
42000 <- 21930
master_enabled = true
```

When switched off:

1. Set Force Mode to 0
2. Set charge power to 0
3. Set discharge power to 0
4. `42000 <- 21947`

This exits remote control cleanly.

## 9. Emergency stop

The `Venus NOT-AUS` button immediately sets output to 0, disables RS485 Remote Control, and resets the local setpoints. It is not a replacement for the battery's own protection functions or for an electrical emergency-stop device. It is a software stop for this project.

## 10. Request tracking and underdelivery

The code stores the most recently requested setpoint and compares it with the measured power.

Example:

```text
req=-700
meas=797
rs485=21930
ctrl=2
dis_reg=700
```

This makes it possible to distinguish quickly between the following cases:

- the setpoint does not reach the ESP
- Force Mode is incorrect
- RS485 Remote Control has been disabled
- the register contains the setpoint, but the battery does not deliver it

The `UNDERDELIVER` diagnostic reports significant deviations after several seconds.

## 11. Heartbeat

Every 30 seconds, the TTGO writes a compact diagnostic line to the log.

The most important fields are:

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

After a Marstek firmware update, the heartbeat is a very useful first check.

## 12. Modbus watchdog

If no Modbus data is received for an extended period:

1. after more than 90 seconds, the RS485 master is switched off and back on once
2. if the bus remains unresponsive, the TTGO is restarted later

Setpoints are reset to 0 before recovery.

## 13. Display

The TTGO display shows locally:

- Modbus OK / error
- Wi-Fi quality
- current setpoint
- SoC
- current charging or discharging power
- colour-coded direction indication

This makes the battery state visible directly on the device, even without Home Assistant.

## 14. Buttons

The two buttons on the T-Display adjust the local setpoint in steps and then call the same Modbus control logic.

> **Note:** The button logic is currently experimental and has not yet been tested in practice on the physical reference device.

## 15. Home Assistant / Node-RED

ESPHome publishes the relevant sensors and Number entities natively to Home Assistant. A control system can therefore be built either with Home Assistant automations or with Node-RED.

The actual energy-management logic, such as zero export, night profiles, minimum SoC, or PV surplus control, is intentionally not hard-coded into this repository. The project provides the robust local interface to the Marstek battery.

## 16. Marstek firmware updates

After a Marstek firmware update, first verify manually that:

1. Modbus reading works
2. `42000` returns `21930` while the master is enabled
3. `42010` can be set to 1 or 2
4. `42020/42021` accept small setpoints
5. actual power follows the setpoint

Only then should automatic control be enabled again.

## Credits

The register mapping and many findings concerning Marstek Modbus come from community work. Particularly important references for this project are:

- ViperRNMC: https://github.com/ViperRNMC/marstek_venus_modbus
- Superduper1969: https://github.com/Superduper1969/MarstekVenus-LilygoRS485
