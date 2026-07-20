# Luba Mower Smart Scheduler — paschdan live install

> This is the **`personal/paschdan-live`** branch. All placeholder entities are replaced with the real ones from paschdan's Home Assistant instance. The scheduling side has been refactored to a **single alternating-dispatch automation** — no scheduler-component, no per-zone windows, just one automation that fires daily at 11:00 (and again when rain clears) and picks the next zone from an `input_select` that flips A/B after each mow.

## What changed vs. earlier versions of this branch

Earlier iterations used [nielsfaber/scheduler-component](https://github.com/nielsfaber/scheduler-component) + a window-based orchestrator automation. That worked but felt over-architected for two zones with strict alternation. This iteration collapses everything into **one automation** (`mower_dispatch`) plus the four rain/night automations that manage `input_boolean.do_not_mow`.

Alternation is enforced by `input_select.next_zone_to_mow` (values: `garten` / `seite`). After a **successful mow** (mower reaches `MODE_READY` for 2 min with `progress ≥ 100`), a companion automation `mower_completion_tracker` flips the select to the other zone. If a dispatch attempt fails (rain, mower busy, already mowed today, spacing guard), the select does NOT flip — the same zone keeps its turn until it genuinely completes. This means rain-interrupted mows retain their alternation slot: the zone that started but didn't finish will be tried again on the next dispatch.

The daily trigger fires at **09:00** — early enough to catch a dry morning window before typical afternoon rain, late enough to avoid dew and neighbor-noise complaints.

## Mower state model

The dispatcher only starts a NEW mow when `sensor.yuka_mnu7nps5_activity_mode == MODE_READY` (mower is docked and idle). All other activity_modes are recognized:

| Activity mode | Interpretation | Dispatcher behavior |
|---|---|---|
| `MODE_READY` | Docked, idle | Ready to start a fresh mow |
| `MODE_WORKING` | Actively mowing | No action; wait for completion |
| `MODE_RETURNING` | Heading back to dock | No action; wait for READY |
| `MODE_PAUSE` | Interrupted (rain, stuck, error) | No action; `automation.mower_paused_continue` handles resumption |

This division of labor means:
- **Our `mower_dispatch`** only starts new mows from READY. Never tries to press an Aufgabe button on top of a paused task.
- **Your independent `mower_paused_continue`** (unrelated to this package) handles resumption of paused tasks by calling `lawn_mower.start_mowing`, with its own 3-retry budget and battery-gate.

Together they cover both paths: fresh starts (us) and resume-after-pause (them).

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
| Alternation state | `input_select.next_zone_to_mow` (defined in this package) |

## Files

- [`packages/luba_mower_scheduler.yaml`](./packages/luba_mower_scheduler.yaml) — helpers, template sensors, rolling-window rain statistics, scripts, and five automations (four rain/night gates + the one `mower_dispatch`). Drop into `<config>/packages/`.
- [`dashboard.yaml`](./dashboard.yaml) — Lovelace view. Paste into a dashboard via raw config editor.

## Dispatch logic

```
┌──────────────────────────────────────────────────────────────────────┐
│  automation.mower_dispatch                                           │
│                                                                      │
│  Triggers:                                                           │
│   • time: 09:00 daily                                                │
│   • state: input_boolean.do_not_mow  on -> off  (rain just cleared)  │
│                                                                      │
│  Guards (all must pass):                                             │
│   • input_boolean.mowing_automation_enabled = on                     │
│   • input_boolean.do_not_mow                = off                    │
│   • sensor.yuka_mnu7nps5_activity_mode      = MODE_READY             │
│   • Before sunset - 3h                                               │
│                                                                      │
│  Runtime checks against the target zone (input_select.next_zone_to_  │
│  mow):                                                               │
│   • Hasn't mowed today                                               │
│   • At least 2 days since same zone last mowed (min spacing)         │
│                                                                      │
│  If all pass:                                                        │
│    -> script.mow_<zone>                                              │
│                                                                      │
│  If any fail: do nothing, next trigger tries again with SAME zone.   │
│  next_zone_to_mow is NOT flipped here — the completion tracker owns  │
│  that.                                                               │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  automation.mower_completion_tracker                                 │
│                                                                      │
│  Trigger:                                                            │
│   • state: activity_mode -> MODE_READY for 2 min                     │
│                                                                      │
│  Conditions:                                                         │
│   • sensor.yuka_mnu7nps5_progress >= 100                             │
│   • input_text.last_lawn_mowed in ['Garten', 'Seite']                │
│     (i.e. our script attributed this mow, we haven't already         │
│     credited it, and it's not a random dock event)                   │
│                                                                      │
│  Action:                                                             │
│   • Flip input_select.next_zone_to_mow to the OTHER of the just-     │
│     completed zone                                                   │
│   • Clear input_text.last_lawn_mowed so we don't re-flip on          │
│     subsequent MODE_READY events (mower re-docks, HA restarts, ...)  │
└──────────────────────────────────────────────────────────────────────┘
```

## Requirements

| Component | Where | Purpose |
|---|---|---|
| [Mammotion HA integration](https://github.com/mikey0000/Mammotion-HA) | HACS integration | Exposes the mower entities |
| A weather integration with hourly precipitation | e.g. built-in Met.no `weather.forecast_home` | Feeds `sensor.rain_today` / `_probability` / `_current_hour` |

**No** scheduler-component or scheduler-card required by this iteration. If you had them installed, they still work but this package doesn't use them.

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

## Rain semantics

Six rain signals gate mowing. Any one being wet turns on `input_boolean.do_not_mow`:

| Signal | Threshold | What it captures |
|---|---|---|
| `sensor.rain_today` | > 3 mm | Total precipitation forecast for the rest of today |
| `sensor.rain_probability` | > 50 % | Fraction of today's forecast hours with any precipitation (proxy for Met.no's missing native probability) |
| `sensor.rain_current_hour` | > 0.5 mm | Precipitation forecast for the current hour |
| `sensor.rain_last_3h` | > 5 mm | Rolling sum: is the grass damp from recent rain? |
| `sensor.rain_last_6h` | > 25 mm | Rolling sum: was there heavy rain earlier today? |
| `weather.forecast_home` | in `[rainy, pouring, lightning-rainy, lightning, hail, snowy, snowy-rainy]` | Actively-wet current weather condition |

The 3h/6h rolling sums are HA's built-in `statistics` platform sensors feeding off `sensor.rain_current_hour`. Damp-grass fix: even after rain stops, the grass stays wet, so we keep blocking until the rolling window ages out.

`reset_do_not_mow_if_dry` requires ALL six signals to be dry AND it's mid-day.

Approach inspired by [simon42.com's Luba-2 automation](https://www.simon42.com/luba-2-mahroboter/#starten-des-luba-2-yuka-maehroboters).

## Manual mow

Dashboard buttons `Garten jetzt mähen` / `Seite jetzt mähen` call `script.mow_garten` / `script.mow_seite` directly. The script has its own guards (redundant with the automation), so pressing the button on a raining day still refuses. Manual mowing does NOT flip `next_zone_to_mow` — that's the automation's job.

## Interaction with `automation.mower_paused_continue`

You already have a separate automation, "Mower - Smart Restart and Reset", which handles stuck retries independently. It watches `sensor.yuka_mnu7nps5_activity_mode` for `MODE_PAUSE` and retries via `lawn_mower.start_mowing` up to 3× per stuck event using `input_number.mower_retry_count`. It doesn't overlap with the dispatcher and stays.

## Adding a zone

1. In `packages/luba_mower_scheduler.yaml`:
   - Add `<zone>_last_mow_1/2/3` blocks under `input_datetime:`.
   - Add a `"<Zone> Days Since Mow"` sensor to the first `template:` list item.
   - Add the zone slug to `input_select.next_zone_to_mow.options`.
   - Add a wrapper script `mow_<zone>` under `script:` that calls `script.mow_lawn` with the zone's `zone_slug` / `zone_button` / `zone_label`.
2. Update `automation.mower_dispatch`'s `other_zone` calculation — with three zones it needs to cycle (garten → seite → third → garten). Simplest: replace the ternary with a lookup dict.
3. Reload scripts + automations (Developer Tools → YAML). Restart HA if you also touched `sensor:` or `template:`.
4. Add cards to `dashboard.yaml` for the new zone.

`script.mow_lawn` itself is already parameterized — no changes needed there.
