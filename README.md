# esphome-lwz-180

English | [Deutsch](README-DE.md)

ESPHome package for the Stiebel Eltron LWZ 180 ventilation unit. The ESP connects as an I2C slave to the control panel bus, listens to the actual values and the display, and publishes them via MQTT discovery as its own Home Assistant device. Via an HA select and an HA switch it can set the fan level and start intensive ventilation.

The analysis of the I2C bus originates from the predecessor project [kdarnok/lwz180](https://github.com/kdarnok/lwz180/), see Credits.

The package runs on any ESP32 with esp-idf and MQTT. Tested: ESP32-S3, ESP-IDF 5.5, ESPHome 2026.10.

## Hardware

- ESP as **hardware I2C slave** on the control panel bus between control panel and LWZ controller (slave address 42, plus general call, no clock stretching).
- Galvanic isolation via a carrier board with **ISO1540**. ESP side: SDA GPIO15, SCL GPIO17 (example, changeable via substitution). The ESP-side pull-ups are not fitted; the internal pull-ups of the ESP32-S3 are sufficient. On the LWZ side, the pull-ups of the LWZ apply.
- The ESP acts as master only briefly when it sends keys (fan level, intensive ventilation). For this it takes the slave down and brings it back up afterwards.

## Installation

Remote, in your own ESPHome YAML:

```yaml
packages:
  lwz_180: github://bastard-GIT/esphome-lwz-180/lwz_180.yaml@main
```

For stable builds, use a tag or commit hash instead of `@main`. A complete example is in `example/lwz_180_example.yaml` (included locally with `!include ../lwz_180.yaml`). `secrets.yaml.example` is also there.

## Requirements

- ESP32 with `esp32: framework: type: esp-idf`.
- `mqtt:` values, discovery and commands run via MQTT, not via the Native API.
- `ota:` with `platform: esphome` and `id: ota_esphome` (or another id, then set `lwz_ota_id`). The package extends this component via `!extend`. The slave is taken down during OTA.

The package sets the required `sdkconfig_options` itself (I2C slave driver V2, `CONFIG_I2C_ISR_IRAM_SAFE`, ring buffer functions in IRAM) and includes its headers from the folder `lwz_180/`.

## Substitutions

Default values, overridable in your own YAML under `substitutions:`.

| Substitution | Default | Meaning |
|---|---|---|
| `lwz_sda_gpio` | `15` | SDA GPIO (number without "GPIO") |
| `lwz_scl_gpio` | `17` | SCL GPIO |
| `lwz_i2c_slave_addr` | `42` | 7-bit address of the ESP as slave |
| `lwz_internal_pullup` | `true` | internal pull-ups |
| `lwz_prefix` | `lwz180` | MQTT and ID prefix of the LWZ values |
| `lwz_device_name` | `Stiebel Eltron LWZ 180` | HA device name |
| `lwz_model` | `LWZ 180` | HA model |
| `lwz_availability_topic` | `""` | empty = availability topic of the MQTT component (default `<topic_prefix>/status`), otherwise own topic, e.g. `heatingpump/status` |
| `lwz_ota_id` | `ota_esphome` | id of the OTA component |

## MQTT topics

| Topic | Content |
|---|---|
| `<prefix>/<wert>/state` | Actual values |
| `<prefix>/availability` | online as long as valid LWZ packets arrive (offline after 15 min without a packet) |
| `<prefix>/luefterstufe/set` | Set fan level: `0` to `2`, `3` = intensive ventilation on |
| `<prefix>/intensivlueftung/set` | `ON` / `OFF` |
| `<prefix>/intensivlueftung/state` | State (retained) |
| `<prefix>/steuerung/result` | Result text of the last command (not retained) |
| `homeassistant/sensor/<prefix>/<wert>/config` | MQTT discovery |

## Home Assistant entities

All belong to one device. Sensors (entities are created when the value is first received):

- Temperatures: Außenlufttemperatur (outside air temperature), Zulufttemperatur (supply air temperature), Ablufttemperatur (exhaust air temperature), Fortlufttemperatur (outgoing air temperature), Taupunkt Abluft (exhaust air dew point), Taupunkt Außenluft (outside air dew point)
- Humidity outside air: Feuchte Außenluft (outside air humidity), Feuchte Abluft (exhaust air humidity)
- Airflow: Volumenstrom Zuluft (supply airflow), Volumenstrom Fortluft (outgoing airflow) (m³/h)
- Zuluftlüfter Ansteuerung (supply fan control), Fortluftlüfter Ansteuerung (outgoing fan control) (%), Zuluftlüfter Drehzahl (supply fan speed), Fortluftlüfter Drehzahl (outgoing fan speed) (rpm)
- Lüfterstufe (fan level) (from the display), Lüfterstufe Einstellung (fan level setting, from type 02; sent only on a level change, 0.4-1.5 s after the display)
- Volumenstrom Soll Zuluft, Volumenstrom Soll Abluft (target airflow, m³/h; confirmed on our bus)
- Fühlerwiderstand Fortluft, Fühlerwiderstand Zuluft (sensor resistance, Ω, PT1000; confirmed on our bus; PR #2 lists 35 as exhaust air, our logs show outgoing air)
- Not yet seen on our bus (from PR #2 of kdarnok/lwz180): Bypass (0/1), Volumenstrom Stufe 0-3 (P6-P9, m³/h), Druck P10-P13 (pressure, Pa)
- Vorheizung Leistung (preheater power), Filterlaufzeit (filter runtime), Betriebsdauer Gerät (device operating hours), Betriebsdauer Lüfter (fan operating hours) (only known from an external source, not yet seen on our bus)

Control:

- Select **Lüfterstufe einstellen** (set fan level) (options 0 to 3)
- Switch **Intensivlüftung** (intensive ventilation)

## Operating the fan level

The ESP presses keys like the control panel and checks the result on the display (closed control loop).

- Level 0 to 2: rotary wheel burst, 5 packets at 100 ms intervals (code `02` = up, `01` = down) produce one level. About every fourth burst has no effect, mostly after a direction change. For this reason the firmware checks the level in the display packet and repeats the burst, at most 8 times, at most 60 s per command.
- Level 3 cannot be reached with the rotary wheel, only as intensive ventilation. The select element "3" starts it.
- Intensive ventilation on: power-vent long press (`08`), after 500 ms `0c`. It runs until the end of the timer. Ending it early is only possible by changing the level; the switch OFF sends one burst for this.
- Keys are sent only when the display shows the basic state, i.e. no menu is open. Otherwise the rotary wheel would scroll through the menu.

## Diagnostics

Topics `stiebel-i2c/*`: `status` (every 5 s), `rx` (sniffer, off by default), `crash`, commands `cmd/sniff`, `cmd/scan`, `cmd/key`. `cmd/key` sends key codes onto the bus, use with caution.

## Credits

The basis for this package is the project [kdarnok/lwz180](https://github.com/kdarnok/lwz180/). It analyzed and documented the I2C bus of the LWZ 180: packet format with CRC, actual values, control panel key codes and the display encoding. Many thanks to kdarnok for this groundwork.

Thanks also to KKSzymanowski for the extended table of actual-value types ([PR #2](https://github.com/kdarnok/lwz180/pull/2)): fan level, target airflow, sensor resistance, bypass, pressures.

This package contains no code from kdarnok/lwz180. It is an independent implementation for ESP32 and ESPHome and uses the findings documented there.

## License

GPL-3.0, see [LICENSE](LICENSE). The I2C bus analysis this package builds on comes from kdarnok/lwz180 (GPL-2.0); no code was taken from it.
