# Luba Mower Smart Scheduler for Home Assistant

A rain-aware, per-zone mowing scheduler for Mammotion robot mowers (Luba / Luba 2 / Luba 3 and similar), built entirely in Home Assistant.

- Schedule each lawn/zone independently (once or twice a week)
- Automatically delays mowing when it is raining or rain is forecast
- Optional night block so the mower never goes out after dusk
- Retries automatically when conditions clear
- Simple dashboard using only built-in cards (no custom card required)
- Live GPS map of the mower, no extra hardware needed

Full write-up and background:
[Luba Mower Home Assistant Scheduler, the 2026 update](https://peterblandford.com/blog/2026/06/02/luba-mower-home-assistant-scheduler-v2/)

---

## 2026 rewrite, please read if you used the old version

This project was rebuilt in June 2026 around the current Mammotion-HA integration (0.6.x). The important change:

- **Old approach:** scripts called `mammotion.start_mow` with `areas: [switch.<mower>_area_*]` and explicit angle/speed parameters.
- **New approach:** scripts press the mower's **saved-task button** for each zone (`button.<mower>_<task>`).

Why the change: on Mammotion-HA 0.6.x, mowing zones are exposed as **buttons** (one per saved task/map area in the Mammotion app), and the old `switch.<mower>_area_*` entities no longer reliably exist, especially on the Luba 3. Pressing the saved-task button runs that task using the settings you saved **in the Mammotion app** (including the cutting angle, which you can see on the app's map). This is more robust across integration updates and easier to tune.

Everything is now generic (`lawn_mower.mower`, zones `lawn_1` / `lawn_2` / `lawn_3`) so you can drop it in and rename to suit. The blade-maintenance and scene helpers from the old version were removed to keep the project focused on scheduling.

---

## Requirements

| Component | Required? | Where |
|---|---|---|
| [Mammotion Home Assistant integration](https://github.com/mikey0000/Mammotion-HA) | Yes | HACS (custom repository) |
| A weather integration providing rainfall + probability | Optional (for rain delay) | e.g. core [OpenWeatherMap](https://www.home-assistant.io/integrations/openweathermap/) |

The example dashboard uses **built-in Lovelace cards only**, so no Mushroom / map-card / camera-agora-card is needed. (You can of course swap in custom cards if you prefer.)

### Before you start: create your mowing tasks in the Mammotion app

This scheduler triggers tasks you have already saved in the Mammotion app. For each lawn/zone:

1. In the Mammotion app, create and save a mowing task for that area (set the cutting angle, speed, blade height, edge mode, etc.).
2. After the integration syncs, that task appears in Home Assistant as a button, e.g. `button.mower_back_lawn`. Find it in **Developer Tools > States** (search `button.`).

If a button does not exist for a zone, the scheduler cannot mow it, create the task in the app first.

---

## Installation

These files use Home Assistant's split-configuration includes. If you already split your config, merge the keys; otherwise add the include lines below.

1. **Copy the YAML files** into your Home Assistant `config/` folder (next to `configuration.yaml`).

2. **Add the includes** to `configuration.yaml` (skip any you already have, and merge instead of duplicating the top-level key):

   ```yaml
   input_boolean:  !include input_boolean.yaml
   input_select:   !include input_select.yaml
   input_datetime: !include input_datetime.yaml
   input_text:     !include input_text.yaml
   template:       !include template.yaml
   script:         !include scripts.yaml
   automation:     !include automations.yaml
   ```

   If you already have, say, `automation: !include automations.yaml`, merge the contents of this file into yours rather than adding a second key.

3. **Restart Home Assistant** (or reload each domain from Developer Tools > YAML).

4. **Rename the placeholder entities** to match your setup. Search-and-replace across `scripts.yaml`, `automations.yaml` and `dashboard.yaml`:
   - `lawn_mower.mower` -> your mower entity (Developer Tools > States, search `lawn_mower.`)
   - `button.mower_lawn_1` / `_lawn_2` / `_lawn_3` -> your saved-task buttons
   - rename the `lawn_1` / `lawn_2` / `lawn_3` helpers if you want friendlier names

5. **Add the dashboard.** Open `dashboard.yaml`, then paste it into a new dashboard view (Settings > Dashboards > Edit > raw configuration editor, under `views:`), and replace the placeholder entities.

6. **Set it running:** turn on **Mowing Automation Enabled**, choose a frequency per zone, and set each zone's **Next Mow** to a future date/time. The scheduler fires at that time and then reschedules itself.

---

## How it works

- Each zone has a **Next Mow** time (`input_datetime.lawn_X_next_time`) and a **Frequency** (`input_select.lawn_X_schedule`).
- A per-zone **Scheduler** automation fires at the Next Mow time. If the scheduler is enabled, the zone is not set to `never`, `do_not_mow` is off, and the mower is idle, it runs `script.mow_lawn_X`, which presses the saved-task button and reschedules the next run (+7 days for once a week, +3 days for twice a week).
- If anything blocks the mow (rain, night, mower busy), the Next Mow time is pushed forward an hour (jumping past dusk overnight), so the zone retries automatically.
- **Rain delay** (optional): `do_not_mow` turns on when `sensor.rain_today` is above 3 mm or `sensor.rain_probability` is above 50%, and clears when it is dry. See `template.yaml` to wire these to your weather integration. With no rain sensors, rain delay simply never triggers.
- **Night block** (optional): with **Block Night Mowing** on, `do_not_mow` turns on an hour before sunset and clears an hour after sunrise.

---

## FAQ

**How do I get the mower's GPS / map? Do I need a tracker or tag?**
No extra hardware. The Mammotion integration already provides a `device_tracker.<mower>` entity from the mower's own GPS. The built-in `map` card in `dashboard.yaml` shows it and auto-centres, no coordinates to configure.

**Where do I get the camera card?**
You don't need a custom card any more. The dashboard uses the built-in `picture-entity` card pointed at your `camera.<mower>` entity (Luba models with a camera). Remove that card if your model has no camera.

**Where exactly do the files go / how do I do the first install?**
See [Installation](#installation) above. The files sit next to `configuration.yaml`, and you reference them with the `!include` lines shown. Restart, then rename the placeholder entities to match yours.

**Can it drive two mowers?**
Yes, as an advanced setup: duplicate the helpers/scripts/automations per mower (or add an `input_select` per zone to choose the mower), and have each zone's scheduler check its assigned mower's state. This project ships single-mower to keep it simple.

---

## Credits

Built on top of the excellent [Mammotion Home Assistant integration](https://github.com/mikey0000/Mammotion-HA) by Michael Arthur. Thanks to everyone who has opened issues and helped improve this.
