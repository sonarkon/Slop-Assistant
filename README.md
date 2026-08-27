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

---

### 2. Onkyo RI - Volume Control via Webhook (Automation Blueprint)

`blueprints/automation/onkyo_volume_control.yaml`

Control an Onkyo receiver's volume with an LG Magic Remote, even when the
TV's audio output is set to optical — which normally blocks the TV's own
volume keys entirely.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fsonarkon%2FSlop-Assistant%2Fmain%2Fblueprints%2Fautomation%2Fonkyo_volume_control.yaml)

**What it does:**

- Listens for two incoming webhooks (volume up / volume down)
- On each webhook, presses the matching button entity on an ESPHome
  Onkyo RI Bridge, which sends the RI command over a wired 3.5mm
  connection to the receiver

**Why this works around the optical lockout:** When a rooted LG webOS TV
uses [lginputhook](https://github.com/Simon34545/lginputhook), button
presses are intercepted at kernel level — before the TV firmware decides to
suppress them because optical audio is selected. The hook fires a `curl`
POST to Home Assistant, which is what this blueprint listens for.

**Requirements:**

- An ESP32 flashed with the included ESPHome config (`esphome/onkyo-ri-bridge.yaml`)
- A rooted LG webOS TV with [lginputhook](https://github.com/Simon34545/lginputhook)
  installed and configured to POST to the two webhook URLs on volume keypress
- The [webOS Homebrew Channel](https://github.com/webosbrew/webos-homebrew-channel)
  for autostart on the TV

**Hardware:**

- ESP32 (any dev board)
- 470Ω resistor
- 3.5mm mono jack → Onkyo RI input

**Setup:**

1. Flash `esphome/onkyo-ri-bridge.yaml` to your ESP32 via the ESPHome
   Dashboard in Home Assistant (first flash: USB, then OTA).
2. Import this blueprint and create an automation — point it at your ESP32's
   volume button entities and set the webhook IDs to match what lginputhook
   is configured to POST to.
3. In lginputhook's web UI (`http://[TV-IP]:1842`), set the volume-up key
   (ID 115) and volume-down key (ID 114) to:
   ```
   curl -s -X POST http://[HA-IP]:8123/api/webhook/onkyo_volume_up
   curl -s -X POST http://[HA-IP]:8123/api/webhook/onkyo_volume_down
   ```

---

### 3. Onkyo RI - Power Sync with TV (Automation Blueprint)

`blueprints/automation/onkyo_power_sync.yaml`

Automatically powers an Onkyo receiver on and off in sync with any TV
integrated in Home Assistant.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fsonarkon%2FSlop-Assistant%2Fmain%2Fblueprints%2Fautomation%2Fonkyo_power_sync.yaml)

**What it does:**

- When the TV turns on → sends RI power-on to the receiver
- When the TV has been off for a configurable delay → sends RI standby
- The delay avoids cutting power during accidental TV restarts

**Requirements:**

- An ESP32 flashed with `esphome/onkyo-ri-bridge.yaml`
- The TV integrated in Home Assistant as a `media_player` entity

**Setup:**

1. Import this blueprint and create an automation.
2. Select your TV's `media_player` entity and the power on/off button
   entities from the Onkyo RI Bridge.
3. Tune the power-off delay to taste (default: 2 minutes).

---

More tools will land here over time — check back for updates.
