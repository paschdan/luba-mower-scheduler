# Luba Mower Smart Scheduler — paschdan live install

> This is the **`personal/paschdan-live`** branch. All placeholder entities are replaced with the real ones from paschdan's Home Assistant instance. Refactored down to **one automation** (`mower_dispatch`) with two trigger-branched responsibilities: start a new mow, and record a completion.

## What changed vs. earlier versions

**Was**: 5–7 automations (scheduler-component schedules + window helpers + retry orchestrator + rain/night gate quartet + dispatcher + completion tracker).

**Is now**: **one** automation. Rain/night state is a template `binary_sensor` computed on the fly. Alternation is derived from timestamps (whichever zone was last mowed longer ago is next). Rain-interrupted mows keep their alternation slot because timestamps only advance on genuine completion.

The daily trigger fires at **09:00** — early enough to catch a dry morning window before typical afternoon rain, late enough to avoid dew and neighbor-noise complaints.

## Wiring in this branch

| Role | Entity |
|---|---|
| Mower | `lawn_mower.yuka_mnu7nps5` (Mammotion Yuka) |
| Zone 1 button ("Garten") | `button.garten_yuka_mnu7nps5_aufgabe_1` |
| Zone 2 button ("Seite") | `button.garten_yuka_mnu7nps5_aufgabe_3` |
| Rain source | `weather.forecast_home` (Met.no), pulled via `weather.get_forecasts` |
| Battery | `sensor.yuka_mnu7nps5_battery` |
| GPS | `device_tracker.yuka_mnu7nps5_yuka_mnu7nps5` |
| Camera | `camera.yuka_mnu7nps5_none` |

## Automation topology

**Ours**: `mower_dispatch` — handles both "start a mow" and "record a completion".

**Yours (pre-existing, unchanged)**: `mower_paused_continue` — resumes stuck-in-PAUSE mows via `lawn_mower.start_mowing` with 3-retry limit.

### `mower_dispatch` — the only automation this package installs

Three triggers, two branches (via `trigger.id`):

```
Triggers:
  id: try
    - time         at: 09:00              (daily attempt)
    - state        binary_sensor.mow_blocked  on -> off   (rain just cleared)
  id: completed
    - state        activity_mode -> MODE_READY  for 2min   (mow finished)

TRY branch runs when trigger.id == 'try':
  guards:
    - mowing_automation_enabled = on
    - binary_sensor.mow_blocked = off
    - activity_mode             = MODE_READY (docked, idle)
    - before sunset - 3h
  runtime:
    - next_zone = whichever of {garten, seite} has the OLDER last_mow_1
    - if spacing_ok (>= 2 days since same zone's last mow):
        run script.mow_<next_zone>

COMPLETED branch runs when trigger.id == 'completed':
  guards:
    - progress >= 100                              (mow actually finished)
    - input_text.last_lawn_mowed in {Garten, Seite}   (we started this mow)
  actions:
    - roll completed zone's last_mow_3 <- 2 <- 1 <- now
    - clear input_text.last_lawn_mowed
```

**Why alternation "just works" from timestamps**:
- `script.mow_lawn` writes `input_text.last_lawn_mowed = "Garten"` but does NOT touch `garten_last_mow_1`.
- Only the COMPLETED branch (MODE_READY + progress 100) bumps `garten_last_mow_1 = now`.
- Rain-interrupted mow: `garten_last_mow_1` stays at the previous successful timestamp. Next dispatch still picks Garten because its timestamp is older.
- Successful mow: Garten's timestamp advances → Seite is now older → next dispatch picks Seite.

### `binary_sensor.mow_blocked` — the on/off gate

Template binary sensor, `true` when ANY of these is true:

| Signal | Threshold | Captures |
|---|---|---|
| `sensor.rain_today` | > 3 mm | Total precipitation forecast for today |
| `sensor.rain_probability` | > 50 % | Fraction of today's forecast hours with rain |
| `sensor.rain_current_hour` | > 0.5 mm | Precipitation forecast for the current hour |
| `sensor.rain_last_3h` | > 5 mm | Rolling-sum: grass damp from recent rain? |
| `sensor.rain_last_6h` | > 25 mm | Rolling-sum: heavy rain earlier today? |
| `weather.forecast_home` | in wet states | Actively-wet current condition |
| night block | `block_night_mowing = on` AND (sun below OR within 1h of sunset) | Optional night-time block |

Companion `sensor.mow_block_reason` shows which specific signal is currently blocking (e.g. `"wet grass (7.2mm last 3h)"`).

## Requirements

| Component | Where | Purpose |
|---|---|---|
| [Mammotion HA integration](https://github.com/mikey0000/Mammotion-HA) | HACS integration | Exposes the mower entities |
| A weather integration with hourly precipitation | e.g. built-in Met.no `weather.forecast_home` | Feeds `sensor.rain_today` / `_probability` / `_current_hour` |

## Install

1. Ensure `configuration.yaml` has:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `packages/luba_mower_scheduler.yaml` to `<config>/packages/luba_mower_scheduler.yaml`.
3. Restart Home Assistant (the legacy `statistics` platform sensors need a restart to load).
4. Paste `dashboard.yaml` into a new dashboard view via **Settings → Dashboards → (your dashboard) → Edit → raw config editor**.
5. Turn on `Mowing Automation Enabled` from the dashboard.

## Manual mow

Dashboard buttons `Garten jetzt mähen` / `Seite jetzt mähen` call `script.mow_garten` / `script.mow_seite` directly. Same guards as `mower_dispatch` apply (mowing enabled, not blocked, mower READY), so pressing the button while raining still refuses. Successful manual mow → same completion attribution as scheduled mow.

## Interaction with `automation.mower_paused_continue`

Your independent automation from before this package. Watches `sensor.yuka_mnu7nps5_activity_mode` for `MODE_PAUSE` and retries via `lawn_mower.start_mowing` up to 3× per stuck event using `input_number.mower_retry_count`. It doesn't overlap with our dispatch:

- Our `mower_dispatch` guard is **`MODE_READY`** — refuses to fire on top of a paused mow.
- Their `mower_paused_continue` handles resumption from PAUSE.

Together they cover both paths: fresh starts (us) and resume-after-pause (them).

## Adding a zone

1. In `packages/luba_mower_scheduler.yaml`:
   - Add `<zone>_last_mow_1/2/3` blocks under `input_datetime:`.
   - Add a `"<Zone> Days Since Mow"` sensor to the first `template:` list item's `sensor:` list.
   - Add a wrapper script `mow_<zone>` under `script:` that calls `script.mow_lawn` with the zone's `zone_slug` / `zone_button` / `zone_label`.
   - In `mower_dispatch`:
     - Update the `next_zone` variable to consider all zone timestamps (e.g. pick the min of `garten_ts`, `seite_ts`, `<new>_ts`).
     - Update the completed-branch template to handle the new zone label.
2. Reload scripts + automations. No HA restart needed unless you also touch `sensor:` (statistics) or `input_datetime:`.
3. Add cards to `dashboard.yaml` for the new zone.

`script.mow_lawn` itself is already parameterized — no changes needed there.

## Legacy helpers still in the YAML

- `input_boolean.do_not_mow` — no longer written by anything.
- `input_text.reason_no_mow` — no longer written by anything.

These are kept in the YAML so HA still loads them (avoids breaking any dashboard rows the user may not have updated yet). Safe to remove them from the YAML once you've confirmed no consumers remain.
