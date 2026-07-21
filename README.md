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

**Ours**: `mower_dispatch` — handles three things: start a new mow, resume a paused mow (after rain clears), and record a completion.

**Yours (pre-existing, unchanged)**: `mower_paused_continue` — retries `lawn_mower.start_mowing` when the mower's `activity_mode` transitions to `MODE_PAUSE` and stays there. This is the "kick a stuck mower" one — it fires on state transitions with `for:` durations (2/5/10/15 min into a fresh pause).

### Division of labor

| Scenario | Who handles it |
|---|---|
| 09:00 daily attempt | `mower_dispatch` (TRY branch, MODE_READY) |
| Rain just cleared, mower idle at dock | `mower_dispatch` (TRY branch, MODE_READY) |
| Rain just cleared, mower still paused at dock | `mower_dispatch` (TRY branch, MODE_PAUSE → resume) |
| Mower stuck mid-lawn, not at dock | `mower_paused_continue` (its retry pattern) |
| Mower mid-mow, rain sensor trips, drives home paused | Yuka firmware docks it; when rain clears, `mower_dispatch` sees MODE_PAUSE + dock and resumes |
| Mower reached dock idle after finishing | `mower_dispatch` (COMPLETED branch — records the completion) |

`mower_dispatch`'s resume branch closes an important gap: `mower_paused_continue`'s triggers are `state → MODE_PAUSE for: 2/5/10/15 min`. If the mower has been in `MODE_PAUSE` for hours (e.g. rain lasted overnight), no new state transition fires and `mower_paused_continue` never re-attempts. Our TRY triggers (09:00 daily + rain-clears) fire on time / event, not on state transitions, so they catch the "stuck overnight" case.

### `mower_dispatch` — the only automation this package installs

Three triggers, three branches (via `trigger.id` + guards):

```
Triggers:
  id: try
    - time         at: 09:00                                (daily attempt)
    - state        binary_sensor.mow_blocked  on -> off     (rain just cleared)
    - state        sensor.yuka_mnu7nps5_battery -> '100'    (finished charging)
  id: completed
    - state        activity_mode -> MODE_READY  for 2min    (mow finished)

Branch A: START fresh mow  (trigger=try, activity_mode=MODE_READY)
  guards:
    - mowing_automation_enabled = on
    - binary_sensor.mow_blocked = off
    - activity_mode             = MODE_READY (docked, idle)
    - input_text.last_lawn_mowed = ''  (no pending completion attribution)
    - before sunset - 3h
  runtime:
    - next_zone = whichever of {garten, seite} has the OLDER last_mow_1
    - if spacing_ok (>= 2 days since same zone's last successful mow):
        run script.mow_<next_zone>

Branch B: RESUME paused mow  (trigger=try, activity_mode=MODE_PAUSE)
  same guards except activity_mode = MODE_PAUSE
  action:
    - lawn_mower.start_mowing on the mower entity
    (Mammotion firmware treats this as "continue paused task", NOT
     "start new task", so the interrupted zone completes.)

Branch C: COMPLETED — record the completion
  guards:
    - progress >= 100                              (mow actually finished)
    - input_text.last_lawn_mowed in {Garten, Seite}   (we started this mow)
  actions:
    - roll completed zone's last_mow_3 <- 2 <- 1 <- now
    - clear input_text.last_lawn_mowed
```

**Why alternation "just works"**:
- **`input_text.last_dispatched_zone`** is the primary alternation signal. Set by `script.mow_lawn` on every dispatch, **never cleared automatically**. Branch A picks the OTHER zone. This guarantees "don't dispatch the same zone twice in a row" even when the previous attempt didn't complete.
- **`input_datetime.<zone>_last_mow_1`** is the "last SUCCESSFUL mow" timestamp. Only advances in Branch C (completion). Used for the 2-day spacing guard and the `days_since_mow` sensor display.
- Fallback: if `last_dispatched_zone` is empty (fresh install), Branch A picks by older timestamp.

**Rain-interrupted example**:
1. `last_dispatched_zone = "Garten"`, Garten mid-mow. Rain interrupts.
2. Rain clears next day. Battery hits 100 → TRY trigger fires.
3. Branch A: `last_dispatched_zone = "Garten"` → **picks Seite**. ✅
4. Seite mows, completes → Branch C bumps `seite_last_mow_1`, clears `last_lawn_mowed`. `last_dispatched_zone` stays "Seite".
5. Next TRY → `last_dispatched_zone = "Seite"` → **picks Garten**. ✅
6. Alternation continues indefinitely.

**Manual override**: set `input_text.last_dispatched_zone` from the UI to force which zone comes next. Setting to "Garten" means Seite is next; "Seite" means Garten is next.

### On completion detection

The COMPLETED trigger fires on `activity_mode` transitioning `MODE_RETURNING → MODE_READY` — that's the successful-completion path (the mower drove home after finishing a task). It fires **instantly** with no `for:` delay because the Yuka's `progress` sensor resets from 100 to 0 within milliseconds of hitting MODE_READY (verified empirically). The `from: MODE_RETURNING` filter also distinguishes real completions from other MODE_READY transitions (manual dock command → MODE_PAUSE → MODE_READY, HA restart, etc).

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

Dashboard buttons `Garten jetzt mähen` / `Seite jetzt mähen` call `script.mow_garten` / `script.mow_seite` directly. Same guards as `mower_dispatch` apply (mowing enabled, not blocked, mower READY), so pressing the button while raining still refuses. Successful manual mow → same completion attribution as scheduled mow, and `last_dispatched_zone` advances so the automation alternates correctly on the next dispatch.

## Cancel a running / paused task

**`button.yuka_mnu7nps5_cancel_current_task`** on the dashboard cancels the current task on the mower. Use this if:

- The mower is mowing the wrong zone and you want to stop it cleanly.
- You want to force it out of `MODE_PAUSE` (e.g. after abandoning a rain-interrupted mow that you don't want the automation to resume).

**`lawn_mower.dock` does NOT cancel the task** — it only sends the mower home and leaves the task in `MODE_PAUSE`. To truly reset the mower to `MODE_READY`, use the cancel button.

The automation's Branch B (RESUME) will re-attempt a paused task as long as the task is still active on the mower. Cancel the task first if you want a clean start.

## Interaction with `automation.mower_paused_continue`

Your independent automation. Watches `sensor.yuka_mnu7nps5_activity_mode` for `MODE_PAUSE` state transitions with `for:` 2/5/10/15 min durations, retries via `lawn_mower.start_mowing` up to 3× per stuck event using `input_number.mower_retry_count`, resets the counter on `MODE_WORKING` for 2min or `MODE_READY`.

**Role**: kick a physically-stuck mower (stuck mid-lawn, wheel jammed, mid-return with error, etc). Fires on freshly-entered PAUSE states.

**Not its role**: resume a mower that's been PAUSE-at-dock for hours. Its state triggers don't re-fire without a new transition. Our `mower_dispatch` Branch B fills that gap by triggering on time/rain-clear events and issuing the same `lawn_mower.start_mowing` when it finds a docked+paused mower.

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
