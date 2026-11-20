# How the controller works (A/C SSC v5.8.4)

The controller is a state machine that runs **every 3 minutes** (plus at sunrise and sunset). The A/C has seven stages, **26 °C (gentlest) down to 20 °C (coldest)**, set through Broadlink IR codes. Everything is decided from the battery, the sun and the room, never from a clock alone.

## Inputs
- `batt_w`: battery power. **Negative = charging** (real surplus), positive = discharging.
- `soc` and the helpers `soc_minimo`, `soc_objetivo` (82 %), `soc_reserva_noche`.
- `solar_total`, `load_total` and `sensor.solar_estado_real` (night / first rays / defensive dawn / low transition / useful transition / partial / firm solar).
- `ac_w`: real A/C power from the CT clamp (the A/C is "on" if > 5 W).
- `alcanza`: *"will it charge today"* from the solar forecast (location, weather, panel orientation).
- `sensor.battery_runtime_hours_at_current_load`: hours the battery lasts at the current load.
- Room temperature from the DS18B20.

## Rules, in the order they are evaluated
1. **Sunset / no sun:** release the lock and stop; at night the stage logic is off.
2. **Lock `aire_pro_bloqueo`:** while the solar day isn't over, the solar controller takes over from the manual "Aire Pro" stepping.
3. **Resync:** if you switched the A/C on by hand, the controller adopts the real stage instead of fighting it.
4. **Manual override:** while on, the controller only watches. It hands control back when solar can sustain the load (battery not discharging, solar ≥ load, or SOC ≥ target with the battery nearly idle). In the morning "comfort" mode, a hand-off keeps the cold stages (20–22) if they are sustainable.
5. **Streaks:** every cycle the battery discharges more than 50 W increments a *warn* streak, and a stronger discharge an *off* streak (both capped at 10, reset when it stops).
6. **SOC protection:** SOC below the minimum with the A/C running → switch it **off**, and check again 20 s later with the CT clamp in case the IR code was missed.
7. **Morning comfort:** before 11:00, in comfort mode and at 20–22 °C, it stays cold only if it is *sustainable*: the battery runtime covers at least 65 % of the hours left until the "confidence hour" (default 9:30), or SOC is at least 12 points above the minimum with ≥ 0.35 kWh forecast for the next hour, or the forecast says the day will refill the battery. After the confidence hour it needs real solar: battery charging, solar ≥ load, or forecast power ≥ load. If the A/C draws ≥ 700 W it climbs one stage within 20–22; below that it holds 20.
8. **Defensive dawn:** before noon with weak sun, it picks the stage from the *ratio* between battery runtime and hours until the confidence hour: ratio ≥ 1 keep, ≥ 0.9 up to 21 °C … down to 26 °C below 0.45 or when SOC is within 0.5 point of the minimum.
9. **Solar start:** if the A/C is off, the day isn't over and the sun is useful (or the battery is charging by 300 W; 1200 W if SOC is near the minimum; or SOC ≥ target with the battery almost idle), it **starts at 26 °C** — or recovers the real stage in comfort mode.
10. **Sustained discharge → step warmer:** two warn cycles → one stage up (e.g. 22 → 23), capped in the morning by the runtime ratio.
11. **Strong discharge at 26 °C → off:** three off cycles → A/C off. If it is after 15:00 and at least two of "battery discharging", "solar < load", "solar < 500 W" are true, the solar day is marked finished and the lock is released.
12. **Real surplus → step colder:** battery charging beyond −350 W, room above 20.5 °C, forecast OK and firm/partial solar → one stage colder.
13. **Probe:** SOC at target, battery nearly idle, no warn streak and forecast OK → one stage colder, to test whether there is more room.
14. **Neutral zone:** between −300 W and +250 W it does nothing.

Every decision writes its reason to `input_text.ac_solar_last_action` (for example `bajando_escalon`, `apagado_por_soc`, `zona_neutra`), so the history explains why the A/C did what it did.

## Related automations
- **Aire Pro (Smart Temp Energy):** the soft start. On switching on it sends 26 °C and steps down as the room temperature crosses 26, 25, … 22 °C; an Echo Dot announces each stage.
- **Aire Pro ON → manual override**, **Solar takes control** (when useful solar returns), **Reset solar day** (at sunrise) and **Fan sync** (the fan follows the A/C).
