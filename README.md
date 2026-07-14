# Luba Mower Smart Scheduler — paschdan live install

> This is the **`personal/paschdan-live`** branch. All placeholder entities are replaced with the real ones from paschdan's Home Assistant instance. The **scheduling side has been rewritten to use [nielsfaber/scheduler-component](https://github.com/nielsfaber/scheduler-component) + scheduler-card** instead of the previous hand-rolled `input_datetime` / `input_select` scheduler automation. If you want the generic template, use [`main`](https://github.com/paschdan/luba-mower-scheduler/tree/main) or the upstream [shawwellpete/luba-mower-scheduler](https://github.com/shawwellpete/luba-mower-scheduler).

## What changed vs. the earlier `personal/paschdan-live`

The pre-scheduler-component version had a per-zone `input_datetime.<zone>_next_time` + `input_select.<zone>_schedule` + one big `automation.luba_mower_scheduler` automation doing the "fire at next_time, then reschedule +3/+7 days" dance.

That's gone. Instead:

- Each zone is a `switch.schedule_*` entity managed by nielsfaber/scheduler-component. You edit weekdays, times, and behavior via the scheduler-card UI (same card you already use for the watering schedule).
- The old scheduler automation is deleted. scheduler-component fires a **per-zone wrapper script** (`script.mow_garten`, `script.mow_seite`) at each configured slot. The wrapper is a thin shim that calls the parameterized `script.mow_lawn` with the zone's hard-coded identity. **Why wrappers?** scheduler-card doesn't let you pass `service_data` in the UI, so a schedule that called `script.mow_lawn` directly would lose its zone params on any card-level edit. Wrappers avoid the problem: the schedule just picks a no-argument script.
- All the "should I mow right now?" gating (`mowing_automation_enabled` / `do_not_mow` / mower busy) moved into `script.mow_lawn` as up-front conditions — so a scheduled slot that fires while it's raining or the mower is docked simply no-ops. The next scheduled slot re-tries naturally, no manual retry logic needed.

## Script topology

```
┌───────────────────────────────────────────────────────────────────────┐
│  scheduler-component schedule  (Mon/Wed/Fri 11:00-18:00 for Garten)   │
│                                                                       │
│  11:00 ─► input_boolean.garten_mow_window = on   (window opens)       │
│  18:00 ─► input_boolean.garten_mow_window = off  (window closes)      │
└───────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│  automation.try_mow_when_window_open                                  │
│                                                                       │
│  Triggers on:                                                         │
│   • garten_mow_window off -> on   (window just opened)                │
│   • seite_mow_window  off -> on   (window just opened)                │
│   • do_not_mow        on -> off   (rain just cleared, mid-window)     │
│                                                                       │
│  Guards:                                                              │
│   • mowing_automation_enabled = on                                    │
│   • do_not_mow = off                                                  │
│   • mower not busy (mowing / returning)                               │
│   • before sunset - 3h                                                │
│                                                                       │
│  For each zone whose window is open AND hasn't mowed today:           │
│    calls script.mow_garten  /  script.mow_seite                       │
└───────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│  script.mow_<zone>  (thin wrapper)                                    │
│    -> script.mow_lawn(zone_slug, zone_button, zone_label)             │
│                                                                       │
│  script.mow_lawn                                                      │
│    up-front guards (redundant safety net):                            │
│      • mowing_automation_enabled = on                                 │
│      • do_not_mow = off                                               │
│      • mower not busy                                                 │
│    then:                                                              │
│      • roll last-mow history                                          │
│      • input_text.last_lawn_mowed = "Garten" / "Seite"                │
│      • button.press on the Aufgabe button                             │
└───────────────────────────────────────────────────────────────────────┘
```

### Why the eligibility-window pattern?

The naive design would have the schedule call the script directly at 11:00. That has a fatal flaw: if it's raining at 11:00 the script no-ops (correct), but nothing tries again later. If the rain clears at 12:00, mowing waits until the next scheduled day.

The window pattern fixes this. The schedule opens a window (11:00-18:00 by default) and the orchestrator automation retries every time conditions become green — either at window-open or whenever `do_not_mow` clears. The "already mowed today?" check on `input_datetime.<zone>_last_mow_1` prevents double-mow.

If the whole window passes under rain, the day is skipped and the next scheduled day tries again — that's the correct behavior. The sunset-3h cutoff prevents late-day mowing.

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
| Garten schedule | `switch.schedule_garten_mahen` (Mon/Wed/Fri) → toggles `input_boolean.garten_mow_window` on 11:00, off 18:00 |
| Seite schedule | `switch.schedule_seite_mahen` (Tue/Thu/Sat) → toggles `input_boolean.seite_mow_window` on 11:00, off 18:00 |

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
5. Create one schedule per zone via the **scheduler-card UI** in the dashboard.

   The schedule should have **two timeslots**, both targeting the zone's `input_boolean.<zone>_mow_window` helper:
   - 11:00-18:00 → `input_boolean.turn_on`
   - 18:00-23:59 → `input_boolean.turn_off`

   Or via `scheduler.add` in **Developer Tools → Actions**:

   ```yaml
   action: scheduler.add
   data:
     name: Garten mähen
     repeat_type: repeat
     weekdays: [mon, wed, fri]
     timeslots:
       - start: "11:00:00"
         stop: "18:00:00"
         actions:
           - service: input_boolean.turn_on
             entity_id: input_boolean.garten_mow_window
       - start: "18:00:00"
         stop: "23:59:00"
         actions:
           - service: input_boolean.turn_off
             entity_id: input_boolean.garten_mow_window
   ```

   Same shape for Seite; different weekdays and target `input_boolean.seite_mow_window`.

   You also need the two eligibility-window helpers created as storage-mode input_booleans (**Settings → Devices & Services → Helpers → Toggle**):
   - `garten_mow_window` — name: "Garten Mow Window"
   - `seite_mow_window` — name: "Seite Mow Window"
6. Paste `dashboard.yaml` into a new dashboard view via **Settings → Dashboards → (your dashboard) → Edit → raw config editor**.
7. Turn on `Mowing Automation Enabled` from the dashboard.

## What each schedule fires

Each `switch.schedule_*` calls the matching wrapper script (`script.mow_garten` or `script.mow_seite`). The wrapper is a no-argument script that in turn calls `script.mow_lawn` with three hard-coded fields:

| Field | Purpose |
|---|---|
| `zone_slug` | Used inside the script to derive helper entity IDs like `input_datetime.<slug>_last_mow_1`. Must match the `input_datetime` naming in this package. |
| `zone_button` | The Mammotion saved-task button to press for this zone. |
| `zone_label` | Human-readable string written to `input_text.last_lawn_mowed`. |

**Why the wrapper indirection?** scheduler-card's UI does not let you pass `service_data` to a `script.*` action. Any edit made in the card UI would drop the params silently. The wrapper scripts sidestep this — schedules just pick a no-argument script by name, and the zone identity lives in the wrapper's own sequence.

## Adding a zone

1. **Create a storage-mode helper** for the new zone's eligibility window: `input_boolean.<zone>_mow_window` (Settings → Devices & Services → Helpers → Toggle).
2. In `packages/luba_mower_scheduler.yaml`:
   - Add `<zone>_last_mow_1/2/3` blocks under `input_datetime:`.
   - Add a `"<Zone> Days Since Mow"` sensor to the first `template:` list item.
   - Add a wrapper script `mow_<zone>` under `script:` that calls `script.mow_lawn` with the zone's `zone_slug` / `zone_button` / `zone_label`.
   - In `automation.try_mow_when_window_open`:
     - Add a `- platform: state` trigger on `input_boolean.<zone>_mow_window` (off → on).
     - Add a `- choose:` action block per zone that runs the wrapper script (mirror the existing Garten/Seite blocks).
3. Reload automations + scripts (Developer Tools → YAML). No HA restart needed if you didn't touch the `sensor:` block.
4. Create the schedule in the scheduler-card UI: two timeslots (on and off) targeting the new window helper.
5. In `dashboard.yaml`: add the new schedule switch to the `include:` list of the `custom:scheduler-card`, add the new window helper to the "Mow Controls" card, and duplicate a Garten/Seite card block for the manual-mow button + days-since-mow sensor.

`script.mow_lawn` itself is already parameterized — no changes needed there.

## Rain semantics

**Six** rain signals gate mowing. Any one being "wet" turns on `input_boolean.do_not_mow` and blocks all schedules + manual buttons:

| Signal | Threshold | What it captures |
|---|---|---|
| `sensor.rain_today` | > 3 mm | Total precipitation forecast for the rest of today |
| `sensor.rain_probability` | > 50 % | Fraction of today's forecast hours with any precipitation (proxy for Met.no's missing native probability) |
| `sensor.rain_current_hour` | > 0.5 mm | Precipitation forecast for the current hour (`forecast[0]`) |
| `sensor.rain_last_3h` | > 5 mm | Rolling sum: is the grass damp from recent rain? |
| `sensor.rain_last_6h` | > 25 mm | Rolling sum: was there heavy rain earlier today? |
| `weather.forecast_home` | in `[rainy, pouring, lightning-rainy, lightning, hail, snowy, snowy-rainy]` | Actively-wet current weather condition |

The **3h / 6h rolling sums** are HA's built-in `statistics` platform sensors feeding off `sensor.rain_current_hour`. That gives us "cumulative recent rainfall" without any custom code. This is the **damp-grass fix**: even after rain stops, the grass stays wet, so we keep blocking until the rolling window ages out.

`reset_do_not_mow_if_dry` requires ALL six signals to be dry before clearing the flag, and only clears during the day. `allow_mowing_at_sunrise` uses the same all-dry check so a wet morning doesn't flip the flag off just because sunrise happened.

Approach inspired by [simon42.com's Luba-2 automation](https://www.simon42.com/luba-2-mahroboter/#starten-des-luba-2-yuka-maehroboters), adjusted for Met.no's hourly forecast shape.

### Grace period

The statistics sensors need up to their full window (3h and 6h respectively) to "warm up" after HA starts. During warmup they show partial sums with a `buffer_usage_ratio` / `age_coverage_ratio` < 1. The rain-delay logic still works during warmup — it just uses whatever data is buffered so far.

### Restart requirement

The `sensor: - platform: statistics` block uses legacy YAML syntax that doesn't hot-reload. Adding or changing those two sensors requires a full HA restart. Everything else in this package reloads via `homeassistant.reload_core_config` targets without a restart.

## Interaction with `automation.mower_paused_continue`

You already have a separate automation, "Mower - Smart Restart and Reset", which handles stuck retries independently. It watches `sensor.yuka_mnu7nps5_activity_mode` for `MODE_PAUSE` and retries via `lawn_mower.start_mowing` up to 3× per stuck event using `input_number.mower_retry_count`. It doesn't overlap with the scheduler-component schedules and stays.
