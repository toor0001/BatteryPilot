# Third-party references and notices

This project was developed with reference to public work from the Marstek community.

## ViperRNMC / marstek_venus_modbus

Repository: https://github.com/ViperRNMC/marstek_venus_modbus

Used as an important reference for Marstek Venus Modbus register addresses, values and control behaviour.

The ViperRNMC project is distributed under the MIT License. Its license notice is reproduced below for attribution and compatibility when material from that project is referenced or adapted:

> MIT License  
> Copyright (c) 2025 Viper
>
> Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## Superduper1969 / MarstekVenus-LilygoRS485

Repository: https://github.com/Superduper1969/MarstekVenus-LilygoRS485

Used as a technical reference and inspiration for an ESPHome/LILYGO based RS485 connection to Marstek Venus devices.

At the time of review, no explicit license file was found in the repository root. For that reason this project does not intentionally redistribute substantial verbatim source code from that repository. The implementation in this repository uses its own control, filtering, watchdog, display and diagnostic logic while sharing unavoidable ESPHome/Modbus concepts and public protocol/register information.

## Code comparison note

A review of the current ESPHome YAML against the two projects above did not identify substantial verbatim copying of application logic. Common elements are mainly:

- standard ESPHome configuration structures,
- Modbus RTU settings,
- Marstek register addresses and command values,
- normal UART/RS485 wiring concepts.

The following parts of this repository are project-specific implementations rather than direct copies from the referenced projects: multi-stage AC-power anti-glitch filtering, median-of-three filtering, SoC jump filtering, request/ack tracking, under-delivery diagnostics, Modbus freshness watchdog/recovery, TFT UI, local setpoint handling and the current heartbeat diagnostics.

This notice is informational and is not legal advice.
