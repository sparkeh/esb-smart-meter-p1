# ESB Networks Smart Meter P1 Port -ESPHome

Reading the customer P1 port on ESB Networks (Ireland) smart meters with an ESP32 and ESPHome. Additionally pushing data to Home Assistant.

ESB started enabling the P1 port in late 2026. There is little public documentation of what it sends, so this repo records a real telegram, decodes every OBIS line, and provides a working ESPHome config.

> **Scope:** observed on one single-phase ESB meter in November 2026. Three-phase meters will very likely add L2/L3 lines. ESB can change the payload with meter firmware updates.

## Summary

| | |
|---|---|
| Protocol | DSMR-style P1, plaintext (not encrypted) |
| Serial | 115200 baud, 8N1, inverted signal |
| Interval | One telegram per second |
| Telegram size | ~700 bytes |
| CRC | Yes (`!XXXX` trailer, CRC16), and ESPHome's stock `dsmr` component validates it |
| Header | `/ESB5\…` |

### What ESB does **not** send

- **No instantaneous voltage** (`1-0:32.7.0`). The meter measures it, since it counts sags and swells, but doesn't publish it.
- **No frequency** (`1-0:14.7.0`).
- **No gas or M-Bus channels.**
- **No 4th tariff register** (`1.8.4`). (This may be related only to my meter)

For voltage and frequency you can use something like a shelly, or any existing power monitoring solution.

## Raw telegram (redacted)

```
/ESB5\XXXXXXXXX_F

0-0:1.0.0(261109204214W)
1-0:1.8.0(2099.159*kWh)
1-0:2.8.0(0765.605*kWh)
1-0:3.8.0(0054.070*kvarh)
1-0:4.8.0(0604.658*kvarh)
1-0:1.8.1(00002006.443*kWh)
1-0:1.8.2(00000089.278*kWh)
1-0:1.8.3(00000003.438*kWh)
0-0:96.7.21(00024)
1-0:32.32.0(00002)
1-0:32.36.0(00000)
1-0:31.7.0(001.6*A)
1-0:1.7.0(0000.002*kW)
1-0:21.7.0(0000.000*kW)
1-0:2.7.0(0000.000*kW)
1-0:22.7.0(0000.000*kW)
1-0:3.7.0(0000.000*kvar)
1-0:23.7.0(0000.000*kvar)
1-0:4.7.0(0000.083*kvar)
1-0:24.7.0(0000.083*kvar)
0-0:42.0.0(<METER LOGICAL DEVICE NAME - 16 bytes, hex-encoded ASCII>)
0-0:96.3.10(1)
!XXXX
```

Redactions: the identifier in the header and the `0-0:42.0.0` value are meter-specific. The CRC is replaced because it no longer matches the edited text.

## OBIS reference

| OBIS code | Description | Unit | Example | ESPHome `dsmr` key |
|---|---|---|---|---|
| `/ESB5\…` | Header: manufacturer / identifier | text | `/ESB5\XXXXXXXXX_F` | `identification` (text_sensor) |
| `0-0:1.0.0` | Timestamp `YYMMDDhhmmss` + `W` (winter, GMT) or `S` (summer, IST) | - | `261109204214W` | `timestamp` (text_sensor) |
| `1-0:1.8.0` | Active energy import, total | kWh | `2099.159` | `energy_delivered_lux` |
| `1-0:2.8.0` | Active energy export, total | kWh | `0765.605` | `energy_returned_lux` |
| `1-0:3.8.0` | Reactive energy import, total | kvarh | `0054.070` | none in stock component; use `dsmr_custom` |
| `1-0:4.8.0` | Reactive energy export, total | kvarh | `0604.658` | none in stock component; use `dsmr_custom` |
| `1-0:1.8.1` | Active energy import, tariff register 1 | kWh | `2006.443` | `energy_delivered_tariff1` |
| `1-0:1.8.2` | Active energy import, tariff register 2 | kWh | `89.278` | `energy_delivered_tariff2` |
| `1-0:1.8.3` | Active energy import, tariff register 3 | kWh | `3.438` | `energy_delivered_tariff3` |
| `0-0:96.7.21` | Number of short power failures | count | `24` | `electricity_failures` |
| `1-0:32.32.0` | Voltage sags, L1 | count | `2` | `voltage_sag_l1` |
| `1-0:32.36.0` | Voltage swells, L1 | count | `0` | `voltage_swell_l1` |
| `1-0:31.7.0` | Instantaneous current, L1 | A | `1.6` | `current_l1` |
| `1-0:1.7.0` | Instantaneous active power import, total | kW | `0.002` | `power_delivered` |
| `1-0:21.7.0` | Instantaneous active power import, L1 | kW | `0.000` | `power_delivered_l1` |
| `1-0:2.7.0` | Instantaneous active power export, total | kW | `0.000` | `power_returned` |
| `1-0:22.7.0` | Instantaneous active power export, L1 | kW | `0.000` | `power_returned_l1` |
| `1-0:3.7.0` | Instantaneous reactive power import, total | kvar | `0.000` | `reactive_power_delivered` |
| `1-0:23.7.0` | Instantaneous reactive power import, L1 | kvar | `0.000` | - |
| `1-0:4.7.0` | Instantaneous reactive power export, total | kvar | `0.083` | `reactive_power_returned` |
| `1-0:24.7.0` | Instantaneous reactive power export, L1 | kvar | `0.083` | - |
| `0-0:42.0.0` | Logical device name (hex-encoded ASCII) | - | *redacted* | - |
| `0-0:96.3.10` | Breaker / disconnector state (`1` = connected) | - | `1` | `electricity_switch_position` (text_sensor) |
| `!XXXX` | End of telegram + CRC16 | - | - | handled internally |

Notes:
- The three tariff registers add up exactly to `1.8.0`.
- **Tariff mapping:** which register is day, night or peak depends on your supplier's time-of-use plan. Check it against a bill before you label them.
- On a single-phase meter, the total and L1 power values are identical.
- Current has 0.1 A resolution and can read noticeably higher than active and reactive power imply. Inverter harmonics are the likely cause, but this isn't confirmed.

## Hardware

- **Connector:** RJ12 (6P6C) in the meter's customer port.
- **Board:** an **ESP32-C3** works when powered from the P1 port alone. A classic ESP32 browned out during OTA updates on P1 power (failing at roughly 45%); if you use one, give it a separate 5 V supply.
- **Data Request:** the port only transmits while the Data Request pin is held high. Most P1 boards tie it to +5 V.
- **Signal inversion:** if your board inverts RX in hardware, leave `inverted` off in the config. If RX is wired directly, set `inverted: true`.
- **Wi-Fi:** meter cupboards are often RF dead zones. Below about -75 dBm, OTA becomes unreliable. If you can't improve coverage, use a board with an external antenna.

## ESPHome config

See [`esb-meter-p1.yaml`](esb-meter-p1.yaml). Tested on ESPHome 2026.10.2.

Points worth knowing:
- The meter reports power in kW and reactive power in kvar. The config multiplies by 1000 to give W and var.
- Home Assistant truncates text states at 255 characters, so the raw telegram text sensor is set `internal`. Read it from the device logs instead.
- ESPHome's per-message log length cuts the telegram off partway, which is why the debug lambda in the config logs it line by line.
- Raw telegram logging costs about 60 ms per telegram and triggers ESPHome's "took a long time" warning. Leave it commented out except when diagnosing.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| No data at all | Data Request pin not high, port not activated by ESB, or wrong RX pin |
| Garbage characters | Wrong inversion setting |
| Some sensors never update | Key mapped to an OBIS code ESB doesn't send (e.g. `current` instead of `current_l1`) |
| OTA fails partway through | Brownout on P1 power (classic ESP32) or weak Wi-Fi |
| `dsmr took a long time` warning | Raw telegram logging enabled |

