# Porsche Taycan Smart Charging for Home Assistant

Solar-first EV charging automation for the Porsche Taycan. Charges only when excess solar is being exported to the grid, or during a cheap off-peak tariff window overnight. Designed to never pull from the grid unintentionally during solar hours.

---

## Features

- **Solar-first charging** — starts only after 5 consecutive minutes of solar export, avoiding false starts from cloud cover
- **Off-peak top-up** — charges overnight (00:00–05:00) from cheap-rate grid electricity
- **85% daily cap** — preserves battery health; configurable
- **100% override** — toggle manually on a dashboard, or automatically via a 🚗 tag in Google Calendar
- **Calendar integration** — add 🚗 to any Google Calendar event and the car will charge to 100% overnight, ready for longer trips
- **Public charging safe** — every automation is gated on `device_tracker.taycan = home`; nothing fires away from home
- **Grid import safeguards** — two layers: stops immediately on unexpected charging, and monitors 90s-averaged grid power to catch cases where the car draws more than available solar
- **Diagnostic sensor** — `sensor.taycan_charging_reason` answers "why is my car charging or not charging right now?" in plain English
- **Notifications** — alerts for safeguard events, 100% override completion, and a daily 07:00 morning summary

---

## Prerequisites

| Integration | Required entities |
|---|---|
| [ha-porscheconnect](https://github.com/CJNE/ha-porscheconnect) (HACS) | `switch.taycan_direct_charging`, `binary_sensor.taycan_charging_active`, `sensor.taycan_state_of_charge`, `sensor.taycan_remaining_range_electric`, `device_tracker.taycan` |
| Your solar inverter | A binary sensor that is `on` when exporting to grid, and a real-time grid power sensor (W) |
| Google Calendar | Two calendar entities (personal + work) — or remove the second if not needed |
| Mobile App | A `notify.*` entity for push notifications |

---

## Installation

### 1. Configure entity IDs

Search and replace the following placeholders throughout `automations.yaml` and `packages/taycan_charging.yaml`:

| Placeholder | Replace with |
|---|---|
| `YOUR_EXPORT_SENSOR` | Binary sensor that is `on` when you are exporting solar (e.g. `binary_sensor.sigen_plant_exporting_to_grid`) |
| `YOUR_GRID_POWER_SENSOR` | 90s averaged grid power sensor — positive = import, negative = export (see below) |
| `YOUR_PHONE` | Your mobile notify entity (e.g. `notify.mobile_app_pixel_8`) |
| `YOUR_PERSONAL_CALENDAR` | Google Calendar entity for personal events (e.g. `calendar.john_gmail_com`) |
| `YOUR_WORK_CALENDAR` | Google Calendar entity for work events (e.g. `calendar.work`) |

### 2. Add the grid power statistics sensor

Add the contents of `configuration_snippets.yaml` to your `configuration.yaml`, replacing `sensor.YOUR_GRID_ACTIVE_POWER` with your inverter's real-time grid power sensor.

### 3. Add the package file

Copy `packages/taycan_charging.yaml` to your packages directory. If you don't use packages, copy the `input_boolean` and `template` blocks into the appropriate sections of your `configuration.yaml`.

Ensure packages are enabled in `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 4. Add the automations

Copy the contents of `automations.yaml` into your `automations.yaml` file, or use `!include` to load it separately.

### 5. Reload

```
Developer Tools → YAML → Reload Automations
Developer Tools → YAML → Reload All (for the package helpers)
```

### 6. Add the override toggle to your dashboard

Add `input_boolean.taycan_full_charge_override` as an entity card to a convenient dashboard.

---

## How it works

### Charging starts when:

| Condition | Detail |
|---|---|
| Solar export sustained for **5 minutes** | Avoids false starts from brief cloud cover |
| **Midnight (00:00)** | Off-peak window begins |
| Car **plugged in during off-peak** | Catches late plug-ins after midnight |
| **Car arrives home** | Starts immediately if solar or off-peak is active |
| **Override toggled on** | Starts immediately if conditions are met |

### Charging stops when:

| Condition | Detail |
|---|---|
| Solar export stops | Immediate stop outside off-peak window |
| **05:00** reached | End of off-peak; stops unless solar is still running |
| SoC reaches **85%** | Default daily cap |
| SoC reaches **100%** | Only when override is active; override resets automatically |

### Safeguards (solar hours only):

| Trigger | Response |
|---|---|
| Charging starts outside a valid window | Immediate stop command → retry after 60s → escalation notification |
| Grid import exceeds 100W (90s average) | Stop command → retry after 60s → escalation notification |

Both safeguards are **inactive during 00:00–05:00** where grid import is intentional.

---

## Calendar integration

Add `🚗` anywhere in a Google Calendar event title when you need a full charge for a longer trip:

- *"Drive to Manchester 🚗"*
- *"🚗 Parents weekend"*

At **22:00** each night, the automation scans both calendars for the next 16 hours. If a tagged event is found, it enables the 100% override and sends a notification. Overnight off-peak charging then fills the battery to 100%, and the override resets automatically.

---

## Diagnostic sensor

`sensor.taycan_charging_reason` provides a live plain-English explanation:

```
Charging · solar export · 62% → 85%
Waiting · no solar, outside off-peak · 71% → 85%
Idle · target 85% reached (85%)
Charging · UNEXPECTED · grid 340W · 22%
```

Attributes include: `soc_pct`, `target_soc_pct`, `charging_active`, `ha_switch_on`, `solar_exporting`, `grid_power_avg_w`, `offpeak_window_active`, `full_charge_override`, `car_at_home`.

---

## Notifications

| Notification | When |
|---|---|
| ⚠️ Unexpected charging stopped | Safeguard fired |
| ⚠️ Grid import safeguard triggered | Car was drawing from grid during solar hours |
| ⚠️ Stop command retried | First stop was ignored |
| 🚨 Cannot stop charging | Both stop attempts failed — manual action needed |
| ✅ Taycan fully charged | Override complete, reached 100% |
| 🚗 Charging to 100% tonight | Calendar tag detected at 22:00 |
| 🚗 Taycan Morning (07:00) | Daily battery %, range, and charging status |

---

## Known limitations

- **Solar start/stop asymmetry** — start requires 5 minutes of sustained export; stop is immediate. On heavily variable solar days the car may never get a 5-minute window to start.
- **No cable-connected sensor** — the Porsche integration doesn't expose cable connection state, only active charging state. The arrived-home automation sends a start command even if the car isn't plugged in (harmless, but creates logbook noise).
- **Porsche API reliability** — commands go via Porsche Connect servers. If the API is unavailable, commands fail silently. The retry logic mitigates this but doesn't guarantee delivery.
- **Off-peak is intentional grid use** — if you have no cheap-rate tariff, remove the `taycan_charge_start_offpeak` automation and the `before: '05:00:00'` time conditions from the safeguards.
- **Device tracker lag** — the Porsche device tracker polls periodically and may lag by a few minutes.

---

## Adjusting the off-peak window

The default off-peak window is **00:00–05:00**. To change it, update these values in `automations.yaml`:

- `taycan_charge_start_offpeak` → trigger time and `before:` condition
- `taycan_charge_stop_end_offpeak` → trigger time
- `taycan_charge_stop_no_solar` → `after:` condition
- `taycan_charge_stop_unexpected_start` → `now().hour >= 5` template
- `taycan_charge_stop_grid_import` → `now().hour >= 5` template
- `taycan_charge_override_on` → `before: '05:00:00'` condition

---

## Adjusting the SoC target

The default daily target is **85%**. To change it, replace `85` in the template conditions across `automations.yaml` and the `packages/taycan_charging.yaml` sensor. Also update the `above: 84` trigger threshold in `taycan_charge_stop_target_reached` (set it to `target - 1`).
