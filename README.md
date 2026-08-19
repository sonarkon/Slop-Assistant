# Slop-Assistant

A personal collection of Home Assistant blueprints, scripts, and tools —
vibe-coded with [Claude](https://claude.ai).

Everything here is built to solve a problem I ran into with my own
smart home setup, then cleaned up and shared in case it's useful to someone
else. 

## License

Everything in this repository is released under the [MIT License](LICENSE)
unless noted otherwise in a specific tool's own folder. Use it, fork it,
change it, ship it — no strings attached.

## Tools

### 1. BILRESA Click Toggle & Smooth Dim (Automation Blueprint)

`blueprints/automation/bilresa-toggle-dim.yaml`

A Home Assistant automation blueprint for the
[IKEA BILRESA scroll wheel](https://github.com/Vituhlos/ha-ikea-bilresa),
built on top of the excellent `ha-ikea-bilresa` custom integration.

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

1. Copy `blueprints/automation/bilresa-toggle-dim.yaml` into
   `config/blueprints/automation/local/` on your Home Assistant instance
   (or import it via *Settings → Automations → Blueprints → Import
   Blueprint* if you're pointing at a hosted URL).
2. Go to *Settings → Automations → Create Automation → Use Blueprint* and
   pick **"IKEA BILRESA - Click Toggle & Smooth Dim"**.
3. Select your wheel's channel `event` entity (e.g.
   `event.bilresa_scroll_wheel_channel_1`) and the light(s) you want to
   control.
4. Optionally tune the brightness-per-notch and transition timings to
   taste.

More tools will land here over time — check back for updates.
