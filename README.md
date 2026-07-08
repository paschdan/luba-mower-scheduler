# Luba Mower Smart Scheduler — paschdan live install

> This is the **`personal/paschdan-live`** branch. All placeholder entities are replaced with the real ones from paschdan's Home Assistant instance. If you want the generic template, use [`main`](https://github.com/paschdan/luba-mower-scheduler/tree/main) or the upstream [shawwellpete/luba-mower-scheduler](https://github.com/shawwellpete/luba-mower-scheduler).

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

## Files

- [`packages/luba_mower_scheduler.yaml`](./packages/luba_mower_scheduler.yaml) — everything (helpers, template sensors, script, automations). Drop into `<config>/packages/`.
- [`dashboard.yaml`](./dashboard.yaml) — Lovelace view. Paste into a dashboard via raw config editor.

## Install

1. Ensure `configuration.yaml` has:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `packages/luba_mower_scheduler.yaml` to `<config>/packages/luba_mower_scheduler.yaml`.
3. Restart Home Assistant.
4. In **Settings > Dashboards**, open the raw editor of your dashboard and paste `dashboard.yaml` under `views:`.
5. Turn on `Mowing Automation Enabled`, pick a schedule per zone (`Garten Schedule` / `Seite Schedule`), and set each zone's `Next Mow` to a future time.

## What's different vs. `main`

| | `main` (generic) | `personal/paschdan-live` (this branch) |
|---|---|---|
| Zone naming | `lawn_1` / `lawn_2` | `garten` / `seite` (matches your mower app tasks) |
| Mower entity | placeholder `lawn_mower.mower` | real `lawn_mower.yuka_mnu7nps5` |
| Button entities | placeholder `button.mower_lawn_N` | `button.garten_yuka_mnu7nps5_aufgabe_1` / `_3` |
| Rain sensors | commented-out OpenWeatherMap template stub | live trigger-based template from `weather.forecast_home` (Met.no) |
| Camera/GPS entities in dashboard | placeholders | real Yuka entities |
| Dashboard labels | English "Lawn 1" / "Lawn 2" | German "Garten" / "Seite" |

## Rain semantics

Met.no's hourly forecast has `precipitation` (mm) but **no** `precipitation_probability` field. To keep the two-signal rain check working, `sensor.rain_probability` is derived as *"fraction of today's remaining forecast hours that have any predicted precipitation, 0-100"*. So a forecast where 3 of the remaining 12 hours today have rain would give `rain_probability: 25`.

The scheduler blocks mowing when EITHER:
- `sensor.rain_today` > 3 mm (predicted rain today, total)
- `sensor.rain_probability` > 50% (more than half of today's remaining hours are wet)

Both signals refresh every 30 minutes and at HA startup.

## Interaction with `automation.mower_paused_continue`

You already have a separate automation, "Mower - Smart Restart and Reset", which handles stuck retries independently. It watches `sensor.yuka_mnu7nps5_activity_mode` for `MODE_PAUSE` and retries via `lawn_mower.start_mowing` up to 3× per stuck event using `input_number.mower_retry_count`.

That automation does not overlap with this scheduler: our scheduler triggers on `input_datetime` time triggers and calls `button.press` on the saved-task button; the stuck-retry automation triggers on activity_mode state changes and calls `lawn_mower.start_mowing`. Both can coexist safely, and together they give you scheduled starts + resilience against pauses/stucks.

## Adding a zone

1. Add `<zone>_schedule` under `input_select:`.
2. Add `<zone>_next_time` + `<zone>_last_mow_1/2/3` under `input_datetime:`.
3. Add a `"<Zone> Days Since Mow"` sensor to the first `template:` list item.
4. In the `luba_mower_scheduler` automation:
   - Add a `- platform: time` trigger with `id: <zone>` and `at: input_datetime.<zone>_next_time`.
   - Add a `<zone>:` entry to `variables.zones` mirroring the existing ones.
5. Add a card block to `dashboard.yaml` calling `script.mow_lawn` with the new zone's `data:` values.

No new script or automation is needed — both are already parameterized.
