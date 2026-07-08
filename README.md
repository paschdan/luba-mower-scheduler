# Luba Mower Smart Scheduler for Home Assistant (packaged)

A rain-aware, per-zone mowing scheduler for Mammotion robot mowers (Luba / Luba 2 / Luba 3 and similar), built entirely in Home Assistant — packaged as **one drop-in YAML** so you only manage one file.

- Schedule each lawn/zone independently (once or twice a week)
- Automatically delays mowing when it is raining or rain is forecast
- Optional night block so the mower never goes out after dusk
- Retries automatically when conditions clear
- Simple dashboard using only built-in cards (no custom card required)
- Live GPS map of the mower, no extra hardware needed

This is a fork of [shawwellpete/luba-mower-scheduler](https://github.com/shawwellpete/luba-mower-scheduler) restructured as a Home Assistant **package**:

- All helpers, template sensors, scripts, and automations live in **one file**: [`packages/luba_mower_scheduler.yaml`](./packages/luba_mower_scheduler.yaml).
- The dashboard remains its own file: [`dashboard.yaml`](./dashboard.yaml).
- The three per-zone schedulers were consolidated into **one automation** using `trigger.id`, and the three mow scripts became **one parameterized `script.mow_lawn`**. Adding a zone is now ~10 lines instead of ~60.
- Ships with **2 zones** (Lawn 1, Lawn 2). See "Adding a zone" below to extend.

The runtime behavior (scheduling, rain delay, night block, retry-past-dusk, history rolling) is identical to upstream. Entity IDs are unchanged for the ones that overlap, so a migration is a straight swap.

Full background on the upstream project:
[Luba Mower Home Assistant Scheduler, the 2026 update](https://peterblandford.com/blog/2026/06/02/luba-mower-home-assistant-scheduler-v2/)

---

## Requirements

| Component | Required? | Where |
|---|---|---|
| [Mammotion Home Assistant integration](https://github.com/mikey0000/Mammotion-HA) | Yes | HACS (custom repository) |
| A weather integration providing rainfall + probability | Optional (for rain delay) | e.g. core [OpenWeatherMap](https://www.home-assistant.io/integrations/openweathermap/) |

The example dashboard uses **built-in Lovelace cards only**, so no Mushroom / map-card / camera-agora-card is needed.

### Before you start: create your mowing tasks in the Mammotion app

This scheduler triggers tasks you have already saved in the Mammotion app. For each lawn/zone:

1. In the Mammotion app, create and save a mowing task for that area (set the cutting angle, speed, blade height, edge mode, etc.).
2. After the integration syncs, that task appears in Home Assistant as a button, e.g. `button.mower_lawn_1`. Find it in **Developer Tools > States** (search `button.`).

If a button does not exist for a zone, the scheduler cannot mow it, create the task in the app first.

---

## Installation

### 1. Enable packages in `configuration.yaml` (once)

If you don't already have this line, add it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 2. Drop the package file in

Copy [`packages/luba_mower_scheduler.yaml`](./packages/luba_mower_scheduler.yaml) into your Home Assistant config:

```
<config>/packages/luba_mower_scheduler.yaml
```

That's it — no other `!include` lines. The package provides all `input_boolean`, `input_select`, `input_datetime`, `input_text`, `template`, `script`, and `automation` entries in one file.

### 3. Rename the placeholder entities

Open `packages/luba_mower_scheduler.yaml` and search-and-replace:

- `lawn_mower.mower` → your mower entity (Developer Tools > States, search `lawn_mower.`)
- `button.mower_lawn_1` → your saved-task button for zone 1
- `button.mower_lawn_2` → your saved-task button for zone 2

Do the same in [`dashboard.yaml`](./dashboard.yaml) for the mower/camera/tracker entities.

### 4. Restart Home Assistant

(or Developer Tools → YAML → **Check Configuration** first if you want a preflight.)

### 5. Add the dashboard

Open [`dashboard.yaml`](./dashboard.yaml) and paste it into a new dashboard view: **Settings → Dashboards → (your dashboard) → Edit → raw configuration editor**, under `views:`.

### 6. Set it running

Turn on **Mowing Automation Enabled**, choose a frequency per zone, and set each zone's **Next Mow** to a future date/time. The scheduler fires at that time and reschedules itself.

---

## How it works

- Each zone has a **Next Mow** time (`input_datetime.lawn_X_next_time`) and a **Frequency** (`input_select.lawn_X_schedule`).
- The single **Luba Mower Scheduler** automation has one `time` trigger per zone (each with a `trigger.id`). When it fires, it looks up the zone's entities from a `variables.zones` map and either runs `script.mow_lawn` (with the zone's parameters) or pushes the next-time forward by 1 hour if something is blocking (rain, night, mower busy). If the retry would land in the dusk window, it jumps to two hours past sunrise instead.
- `script.mow_lawn` presses the zone's saved-task button, rolls the last-mow history, and reschedules the next run based on the zone's frequency (+7 days for once a week, +3 days for twice a week).
- **Rain delay** (optional): `do_not_mow` turns on when `sensor.rain_today` > 3 mm or `sensor.rain_probability` > 50%, and clears when it is dry. Wire the two commented-out template sensors at the bottom of the package to your weather integration to enable this. With no rain sensors, rain delay simply never triggers.
- **Night block** (optional): with **Block Night Mowing** on, `do_not_mow` turns on an hour before sunset and clears an hour after sunrise.

---

## Adding a zone

For each new zone `N`:

1. In [`packages/luba_mower_scheduler.yaml`](./packages/luba_mower_scheduler.yaml):
   - Add a `lawn_N_schedule` block under `input_select:`.
   - Add `lawn_N_next_time` + `lawn_N_last_mow_1/2/3` blocks under `input_datetime:`.
   - Add a `"Lawn N Days Since Mow"` sensor under the first `template:` list item.
   - Under the `luba_mower_scheduler` automation:
     - Add another `- platform: time` trigger pointing at `input_datetime.lawn_N_next_time`, with `id: lawn_N`.
     - Add a `lawn_N:` entry to `variables.zones` mirroring the existing entries.
2. In [`dashboard.yaml`](./dashboard.yaml): copy any `Lawn N` `entities` card and update entity IDs + the `script.mow_lawn` `data:` block.

No new script or automation is needed — both are already parameterized.

---

## FAQ

**How do I get the mower's GPS / map? Do I need a tracker or tag?**
No extra hardware. The Mammotion integration already provides a `device_tracker.<mower>` entity from the mower's own GPS. The built-in `map` card in `dashboard.yaml` shows it and auto-centres, no coordinates to configure.

**Where do I get the camera card?**
You don't need a custom card any more. The dashboard uses the built-in `picture-entity` card pointed at your `camera.<mower>` entity (Luba models with a camera). Remove that card if your model has no camera.

**Can it drive two mowers?**
Yes, as an advanced setup: duplicate the helpers/scripts/automations per mower (or add an `input_select` per zone to choose the mower), and have each zone's scheduler check its assigned mower's state. This project ships single-mower to keep it simple.

---

## Credits

Forked from [shawwellpete/luba-mower-scheduler](https://github.com/shawwellpete/luba-mower-scheduler) — original design and logic by Peter Blandford.

Built on top of the excellent [Mammotion Home Assistant integration](https://github.com/mikey0000/Mammotion-HA) by Michael Arthur.
