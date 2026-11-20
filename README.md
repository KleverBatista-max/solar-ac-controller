# Solar A/C controller (Home Assistant)

Uses surplus solar energy to **pre-cool a room with the air conditioner during the day**, so the battery isn't drained at night. The controller (`A/C SSC v5.8.4`, every 3 min) steps the setpoint (26 → 20 °C) by real room temperature and available solar, with manual override, a defensive dawn mode, warning/off streaks if the battery discharges, and hand‑off at sunset. The A/C is driven by infrared codes through a Broadlink RM mini 3.

Pre-cooling is not blind: the controller also reads the **solar forecast** (remaining energy today, expected power now and the next hour) and only pushes the room colder when the forecast says there is enough sun to refill the battery (`Alcanza a cargar hoy`), or the sun is already covering the load.

## "Aire Pro": soft start, one step at a time
The A/C never jumps straight to its coldest setting. When it is switched on, the controller sends the **26 °C** code, and then goes down stage by stage (25 → 24 → … → 20 °C) as the measured room temperature crosses each threshold, again through IR. That keeps the compressor from a big power spike, which matters when the energy comes from a battery. On the solar side, the SSC controller only lets the stage go lower when the battery is charging with real surplus and the forecast says the day can still refill it; if the battery starts discharging it holds, warns and finally switches off. An Echo Dot announces each stage through Alexa routines. Power is measured by a CT clamp on the A/C (ESP32) and integrated into kWh, so you can chart how much it consumed at each stage. (The air-conditioning side has been switched off since January, so recent data may be empty.)

| Path | What |
|---|---|
| `homeassistant/packages/solar_ac_controller.yaml` | The 6 automations (SSC controller, smart temperature, manual override, solar takes control, daily reset, fan sync), the debug sensor, the A/C energy integration and all helpers |
| `esphome/airconditionermultisensor.yaml` | ESP32 with a CT clamp (A/C power) and a DS18B20 room temperature probe |
| `broadlink/ir_codes.json` | IR codes learned from the A/C (OFF, 20–26 °C) |
| `raspberry-original/solar_air_controller.py` | First version in Python on a Raspberry Pi |

## The air-conditioner ESP32 (`esphome/airconditionermultisensor.yaml`)
- **Current:** a CT clamp on the A/C wire read by the ADC (GPIO34) every 200 ms, smoothed with a 30-sample moving average, calibrated linearly, and anything under 0.2 A is forced to 0 so noise doesn't look like consumption.
- **Power:** an *estimate*, not a real meter: current × 120 V when it is under 1 A and × 220 V above that (no power factor). Good enough to tell "off / running / working hard", which is what the controller needs (it uses > 5 W as "on" and 700 W as "working hard").
- **Temperature:** a DS18B20 on the 1-Wire bus (GPIO4), every 15 s.

## Requires
- **[solar-inverter-modbus-monitor](https://github.com/KleverBatista-max/solar-inverter-modbus-monitor)** provides the solar/battery sensors and SOC helpers used here (`sensor.solar_total_power`, `sensor.solar_estado_real`, `input_number.soc_minimo`…).
- Broadlink integration and SmartIR (HACS). Adapt entity and device IDs to your setup.
- Optional: [bluetti-solar-charger](https://github.com/KleverBatista-max/bluetti-solar-charger) for the second consumer of surplus.

Full logic: [`docs/how-it-works.md`](docs/how-it-works.md). Overview: [solar-home-overview](https://github.com/KleverBatista-max/solar-home-overview).

Created by **John Batista** ([@KleverBatista-max](https://github.com/KleverBatista-max)). Built with AI assistants (ChatGPT, Claude) as tools. MIT licensed. Secrets are not included; replace `!secret` values and regenerate ESPHome keys.
