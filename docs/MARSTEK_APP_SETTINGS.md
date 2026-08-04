# Marstek app settings for this RS485 controller

This project controls the Marstek Venus directly through **Modbus RTU over the physical RS485 connection**. It does not use Marstek's UDP/Local API as its control path.

## Recommended settings for the tested V148 setup

For the reference setup used by this project:

- **Local API / Open API does not need to be enabled.** The ESPHome controller communicates directly over RS485.
- Set the Marstek operating mode to **Manual** before handing charge/discharge control to the external RS485 automation.
- Avoid changing the operating mode from the Marstek app while RS485 external control is active. Firmware behaviour can disable/reset RS485 control when modes are changed.
- The actual charging and discharging decision is made by Home Assistant / Node-RED (or another higher-level controller) through the entities exposed by this project.

## What about a Marstek smart meter?

This setup deliberately replaces the Marstek's own automatic energy-management decision with an external controller.

A Marstek smart meter may still be physically present and its measurements may still be useful, but it should **not be expected to control battery charge/discharge while the battery is being force-controlled through this RS485 setup**. In Manual/force-control operation, your external automation must decide when and how much the battery charges or discharges.

In other words: if you use this project as intended, you are taking responsibility for the control loop yourself. Your automation should therefore also prevent unwanted behaviour such as the battery discharging into an EV charger, immersion heater or another flexible load.

## Local API vs. RS485

These are separate interfaces:

- **Local API / Open API:** network/UDP based Marstek interface.
- **This project:** direct **RS485 / Modbus RTU** interface.

Enabling the Local API is therefore not a prerequisite for this project.

## Firmware note

Marstek firmware behaviour has changed between releases. The reference system was tested with **Venus E Gen3 firmware V148**. Community reports and ViperRNMC discussions show that mode changes can interact with or disable external RS485 control. Re-test the behaviour after firmware updates.

This document describes the tested/reference operating model of this repository, not every possible Marstek configuration.
