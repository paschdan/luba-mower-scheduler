# Luba Mower Smart Scheduler — paschdan live install

> This is the **`personal/paschdan-live`** branch. All placeholder entities are replaced with the real ones from paschdan's Home Assistant instance. The **scheduling side has been rewritten to use [nielsfaber/scheduler-component](https://github.com/nielsfaber/scheduler-component) + scheduler-card** instead of the previous hand-rolled `input_datetime` / `input_select` scheduler automation. If you want the generic template, use [`main`](https://github.com/paschdan/luba-mower-scheduler/tree/main) or the upstream [shawwellpete/luba-mower-scheduler](https://github.com/shawwellpete/luba-mower-scheduler).

## What changed vs. the earlier `personal/paschdan-live`

The pre-scheduler-component version had a per-zone `input_datetime.<zone>_next_time` + `input_select.<zone>_schedule` + one big `automation.luba_mower_scheduler` automation doing the "fire at next_time, then reschedule +3/+7 days" dance.

That's gone. Instead:

- Each zone is a `switch.schedule_*` entity managed by nielsfaber/scheduler-component. You edit weekdays, times, and behavior via the scheduler-card UI (same card you already use for the watering schedule).
- The old scheduler automation is deleted. scheduler-component fires `script.mow_lawn` directly at each configured slot.
- All the "should I mow right now?" gating (`mowing_automation_enabled` / `do_not_mow` / mower busy) moved into `script.mow_lawn` as up-front conditions — so a scheduled slot that fires while it's raining or the mower is docked simply no-ops. The next scheduled slot re-tries naturally, no manual retry logic needed.

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
| Garten schedule | `switch.schedule_garten_mahen` (Mon/Wed/Fri 09:00, set via `scheduler.add`) |
| Seite schedule | `switch.schedule_seite_mahen` (Tue/Sat 09:00, set via `scheduler.add`) |

## Requirements

| Component | Where | Purpose |
|---|---|---|
| [Mammotion HA integration](https://github.com/mikey0000/Mammotion-HA) | HACS integration | Exposes the mower entities |
| [nielsfaber/scheduler-component](https://github.com/nielsfaber/scheduler-component) | HACS integration | Owns per-zone timing (schedules → script) |
| [nielsfaber/scheduler-card](https://github.com/nielsfaber/scheduler-card) | HACS Lovelace | Nice UI to edit schedules — dashboard.yaml uses `custom:scheduler-card` |
| A weather integration with hourly precipitation | e.g. built-in Met.no `weather.forecast_home` | Feeds `sensor.rain_today` / `sensor.rain_probability` |

## Files

- [`packages/luba_mower_scheduler.yaml`](./packages/luba_mower_scheduler.yaml) — helpers, template sensors, script, rain/night automations. Drop into `<config>/packages/`.
- [`dashboard.yaml`](./dashboard.yaml) — Lovelace view including the scheduler-card. Paste into a dashboard via raw config editor.

## Install

1. Ensure `configuration.yaml` has:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `packages/luba_mower_scheduler.yaml` to `<config>/packages/luba_mower_scheduler.yaml`.
3. Install `nielsfaber/scheduler-component` and `nielsfaber/scheduler-card` via HACS if not already there. Add the "Scheduler" integration under **Settings → Devices & Services**.
4. Restart Home Assistant.
5. Create one schedule per zone. Either via the scheduler-card UI in the dashboard, or via `scheduler.add` in **Developer Tools → Actions**:

   ```yaml
   action: scheduler.add
   data:
     name: Garten mähen
     repeat_type: repeat
     weekdays: [mon, wed, fri]
     timeslots:
       - start: "09:00:00"
         stop: "09:15:00"
         actions:
           - service: script.mow_lawn
             entity_id: script.mow_lawn
             service_data:
               zone_slug: garten
               zone_button: button.garten_yuka_mnu7nps5_aufgabe_1
               zone_label: Garten
   ```

   (Same shape for Seite; different weekdays and different `zone_slug`/`zone_button`/`zone_label`.)
6. Paste `dashboard.yaml` into a new dashboard view via **Settings → Dashboards → (your dashboard) → Edit → raw config editor**.
7. Turn on `Mowing Automation Enabled` from the dashboard.

## What each schedule fires

Each `switch.schedule_*` calls `script.mow_lawn` with three fields:

| Field | Purpose |
|---|---|
| `zone_slug` | Used inside the script to derive helper entity IDs like `input_datetime.<slug>_last_mow_1`. Must match the `input_datetime` naming in this package. |
| `zone_button` | The Mammotion saved-task button to press for this zone. |
| `zone_label` | Human-readable string written to `input_text.last_lawn_mowed`. |

## Rain semantics

Met.no's hourly forecast has `precipitation` (mm) but **no** `precipitation_probability` field. To keep the two-signal rain check working, `sensor.rain_probability` is derived as *"fraction of today's remaining forecast hours that have any predicted precipitation, 0–100"*. So a forecast where 3 of the remaining 12 hours today have rain would give `rain_probability: 25`.

Both signals refresh every 30 minutes and at HA startup. The four rain/night automations gate `input_boolean.do_not_mow`, and `script.mow_lawn` short-circuits when `do_not_mow` is on — so the scheduler-component slot fires, the script no-ops, and the next slot re-tries naturally.

## Interaction with `automation.mower_paused_continue`

You already have a separate automation, "Mower - Smart Restart and Reset", which handles stuck retries independently. It watches `sensor.yuka_mnu7nps5_activity_mode` for `MODE_PAUSE` and retries via `lawn_mower.start_mowing` up to 3× per stuck event using `input_number.mower_retry_count`. It doesn't overlap with the scheduler-component schedules and stays.

## Adding a zone

1. In `packages/luba_mower_scheduler.yaml`:
   - Add `<zone>_last_mow_1/2/3` blocks under `input_datetime:`.
   - Add a `"<Zone> Days Since Mow"` sensor to the first `template:` list item.
2. Create the schedule via scheduler-card UI or `scheduler.add` (see step 5 above).
3. In `dashboard.yaml`: add the new schedule switch to the `include:` list of the `custom:scheduler-card`, and duplicate a Garten/Seite card block if you want the days-since-mow + manual-mow button pair.

No new script or automation is needed — `script.mow_lawn` is already parameterized.
