# ESPHome Camper LTE Monitor

Remote monitoring for a campervan, motorhome or boat in storage: a LilyGO
T-A7670E (ESP32 with LTE modem) running ESPHome wakes from deep sleep, reads
Victron, BMS, gas level and temperature sensors via Bluetooth, sends everything
over LTE through a WireGuard tunnel to Home Assistant at home and goes back to
sleep. No Cerbo GX or router has to stay on.

This is a hobby project, not a product. Everything is tailored to my vehicle and
has to be adapted. No support, use at your own risk.

Code comments and log messages are in English. Entity names, notification
texts and dashboard labels are still German, because they are what my own
installation uses; translate them if you like (entity IDs are derived from
the names, so change them consistently).

## Contents

| File | Purpose |
|---|---|
| `esphome/poessl-monitor.yaml` | Firmware for the LilyGO |
| `esphome/secrets.example.yaml` | Template for `secrets.yaml` |
| `homeassistant/poessl_monitor.yaml` | Package: webhook, sensors from the monitor, alerts, weekly report, notification script |
| `homeassistant/poessl_fallback.yaml` | Package: dashboard sensors that switch between Cerbo GX and monitor |
| `homeassistant/dashboards/*.yaml` | Dashboard views (power, information, messages) |

File and entity names still carry the prefix `poessl`/`possl` from my vehicle
(a Pössl van). Replace it with search and replace across all files at once if
you like.

## How a cycle works

1. Wake up, read the shunt current (idle current, before the modem starts)
2. Power up the modem, send `ATE0` (see pitfalls), connect to LTE
3. WireGuard tunnel to the router at home (a Fritz!Box in my case)
4. Bluetooth: Victron (Instant Readout), BMS (active query), Mopeka, Inkbird
5. POST a JSON to a Home Assistant webhook
6. Modem off (`AT+CPOF`), deep sleep

Intervals (set in `substitutions`): 30 min, 120 min at night, 15 min when the
battery is low. A cycle takes about 40 s.

If a run ends in a brownout, crash or watchdog reset, a guard skips the next
cycle instead of restarting immediately. This prevents a boot loop that would
drain the battery. Home Assistant reports the event on the next contact.

## Hardware

- LilyGO T-A7670E (the version without GPS is enough), 5 V supply
- SIM: 1NCE in my case (500 MB / 10 years); about 2–3 kB per report
- Read via Bluetooth in my setup: Victron SmartShunt, Victron SmartSolar MPPT,
  CBT Power BMS, Mopeka Standard (gas), four Inkbird IBS-TH1 mini
- Idle draw of the whole system (Victron devices, BMS, monitor): about 2 W

## Setup

### ESPHome
1. Copy `secrets.example.yaml` to `secrets.yaml` and fill it in.
2. In `poessl-monitor.yaml`, replace the placeholder MACs `AA:BB:CC:DD:EE:0x`
   with your own (BMS in `substitutions`, Mopeka and Inkbird further down).
3. Adjust `peer_allowed_ips` in the WireGuard block to your home network.
4. Remove sensors you don't have, including their lines in the JSON block
   (`root["..."]`), otherwise the build fails.
5. Flash via USB: `esphome run poessl-monitor.yaml`. The device has no WiFi
   and no OTA.

Important: the modem component comes from the open pull request
[esphome#6721](https://github.com/esphome/esphome/pull/6721) and is pinned to a
fixed commit (`external_components`). It does not work with stock ESPHome.
Victron support via
[esphome-victron_ble](https://github.com/Fabian-Schmidt/esphome-victron_ble).

### BMS
The query is written for a CBT Power BMS, based on the protocol in the Python
library [aiobmsble](https://github.com/patman15/aiobmsble) (module
`cbtpwr_bms`). For a different BMS, rewrite the `ble_client` block and the
`bms_rx` sensor based on the matching aiobmsble module. AI assistance
(e.g. Claude) works well for this. If you don't want BMS data, remove the BMS
parts entirely.

### Home Assistant
1. Enable packages in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `poessl_monitor.yaml` and `poessl_fallback.yaml` to `packages/`.
3. In `poessl_monitor.yaml`:
   - set `webhook_id` to your own, non-guessable ID (same ID in the ESP's
     `secrets.yaml`)
   - replace `notify.mobile_app_HANDY_1/2` with your notify services
   - `DEINE_PANDORA_ID`: entities of my Pandora car alarm (starter battery,
     doors). Remove these parts if you don't have one.
4. In `poessl_fallback.yaml`, adjust the Cerbo entities. They come from the
   Victron MQTT integration, and the numbers (e.g. `solarcharger_278`,
   `battery_279`) are the VRM instances of my devices. Without a Cerbo, remove
   the Cerbo branches; only the monitor values are used then.
5. Dashboards: paste the views via "Raw configuration editor". Required HACS
   cards: power-flow-card-plus, card-mod, weather-card.

### Switching between Cerbo GX and monitor
`binary_sensor.possl_cerbo_live` is on if any `sensor.victron_mqtt_*` has
**changed** within the last 15 minutes. It checks `last_changed`, not
`last_reported`, because the integration keeps reporting old values long after
the Cerbo is off. When live, the fallback sensors use the Cerbo values,
otherwise the monitor values as long as its last report is younger than
2 h 15 min. Solar yield uses the maximum of both sources, since the Cerbo
reports 0 after a restart.

## Pitfalls

- **AT echo:** the A7670E ships with `ATE1`. The component's mode detection
  takes the echoed packet for data mode and blocks for 20 s. Fix: send `ATE0`
  in every cycle before `modem.enable`. The A7670E rejects `AT&W`, so the echo
  can't be disabled permanently.
- **Modem during sleep:** when powered via VBUS, GPIO12 does not cut power to
  the modem, hence `AT+CPOF`. Hold the reset and PWRKEY pins LOW during sleep.
- **WireGuard** needs a valid time (SNTP over LTE).
- **Idle current** is measured before the modem starts. Otherwise you only see
  the awake state.
- **Mopeka:** adjust the calibration (`CUSTOM`, full/empty in mm, propane
  ratio) to your own cylinder.

## License

MIT, see [LICENSE](LICENSE).
