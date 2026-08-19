# Slop-Assistant

A personal collection of Home Assistant blueprints, scripts, and tools —
vibe-coded with [Claude](https://claude.ai).

Everything here is built to solve a real problem I ran into with my own
smart home setup, then cleaned up and shared in case it's useful to someone
else. 

## License

Everything in this repository is released under the [MIT License](LICENSE)
unless noted otherwise in a specific tool's own folder. Use it, fork it,
change it, ship it — no strings attached.

## Tools

### 1. BILRESA Click Toggle & Smooth Dim (Automation Blueprint)

`blueprints/automation/bilresa_toggle_dim_blueprint.yaml`

A Home Assistant automation blueprint for the
[IKEA BILRESA scroll wheel](https://github.com/Vituhlos/ha-ikea-bilresa),
built on top of the excellent `ha-ikea-bilresa` custom integration.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fsonarkon%2Fslop_assistant%2Fmain%2Fblueprints%2Fautomation%2Fbilresa_toggle_dim_blueprint.yaml)

**What it does:**

- **Click** the wheel to toggle one or more lights on/off. Turning lights
  back on restores each light's last known brightness — no jarring jump to
  a fixed value.
- **Scroll** the wheel to smoothly dim the lights up or down. Each notch of
  the wheel steps the brightness by a configurable percentage, and every
  targeted light is dimmed relative to its *own* current brightness, so
  lights that started at different levels keep their relative difference.

**Why a blueprint instead of copy-pasting YAML:** the wheel's built-in
`event` entities fire per-channel, batched every ~500 ms–1 s. Getting a
truly reliable toggle + smooth-dim automation out of that took a fair bit
of iterating — using the dedicated `event.received` trigger instead of a
generic `state` trigger, reading `notches` straight from the trigger
context instead of polling live state, and letting each light step
relative to its own brightness instead of forcing every light to a shared
value. The blueprint bakes all of that in, so you just point it at your
wheel channel and your lights.

**Requirements:**

- The [`ha-ikea-bilresa`](https://github.com/Vituhlos/ha-ikea-bilresa)
  custom integration (HACS or manual install), commissioned via Matter.
- Home Assistant recent enough to support the `event.received` trigger
  (2024.10+).

**Setup:**

1. Click the **Import Blueprint** badge above (it pre-fills the blueprint's
   raw URL into your own Home Assistant instance's import dialog), or
   copy `blueprints/automation/bilresa_toggle_dim_blueprint.yaml` manually
   into `config/blueprints/automation/local/` on your Home Assistant
   instance.
2. Go to *Settings → Automations → Create Automation → Use Blueprint* and
   pick **"IKEA BILRESA - Click Toggle & Smooth Dim"**.
3. Select your wheel's channel `event` entity (e.g.
   `event.bilresa_scroll_wheel_channel_1`) and the light(s) you want to
   control.
4. Optionally tune the brightness-per-notch and transition timings to
   taste.

More tools will land here over time — check back for updates.
