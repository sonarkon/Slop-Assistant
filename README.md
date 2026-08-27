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

### 2. Onkyo RI Bridge

Control an Onkyo receiver over its wired RI bus from Home Assistant — volume,
power, and input switching. Built around an ESP32 connected to the receiver's
3.5mm RI input and two Home Assistant blueprints that wire everything together.

**The problem it solves:** When an LG TV's audio output is set to optical, the
Magic Remote's volume keys are silently suppressed by the TV firmware. And there's
no built-in way to power-cycle a dumb receiver in sync with the TV. This toolset
fixes both.

**How it works end-to-end:**

```
Magic Remote (volume key)
  → lginputhook on rooted LG webOS TV   ← intercepts at kernel level, before firmware lockout
  → curl POST to HA webhook
  → HA automation (blueprint)
  → ESPHome button entity
  → ESP32 RI Bridge
  → Onkyo RI bus (3.5mm wired)
  → Onkyo receiver
```

Power sync follows the TV's `media_player` state directly — no remote involved.

---

#### ESPHome firmware

`esphome/onkyo-ri-bridge.yaml`

Flash this to any ESP32 dev board. It exposes five button entities in Home
Assistant: Volume Up, Volume Down, Power On, Power Off, and Input Optical.

**Wiring:**

```
ESP32 GPIO4 → 470Ω resistor → 3.5mm mono jack tip → Onkyo RI input
ESP32 GND   →                  3.5mm mono jack sleeve
```

The RI bus is a 5V DC signal. The ESP32's 3.3V output works in practice;
for a cleaner signal add an NPN transistor (e.g. 2N2222) as a level shifter.

**RI command codes used** (community-verified, tested on TX-SR603 / TX-8020):

| Function      | Code    |
|---------------|---------|
| Volume Up     | `0x1A2` |
| Volume Down   | `0x1A3` |
| Power On      | `0x2F`  |
| Power Off     | `0x420` |
| Input Optical | `0x120` |

> The Input Optical code may vary by model. If `0x120` doesn't work on your
> receiver, try `0x70` or `0x170`.

**First flash:** USB via the ESPHome Dashboard in Home Assistant. All subsequent
updates are OTA over Wi-Fi.

---

#### Blueprint: Volume Control via Webhook

`blueprints/automation/onkyo_volume_control.yaml`

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fsonarkon%2FSlop-Assistant%2Fmain%2Fblueprints%2Fautomation%2Fonkyo_volume_control.yaml)

Listens for two incoming webhooks and presses the matching ESPHome button on
each hit. One automation handles both directions.

**Requires:** ESP32 with `onkyo-ri-bridge.yaml` + rooted LG webOS TV with
[lginputhook](https://github.com/Simon34545/lginputhook) (autostart via
[webOS Homebrew Channel](https://github.com/webosbrew/webos-homebrew-channel)).

**Setup:**

1. Import the blueprint and create an automation. Point it at your ESP32's
   volume button entities. Leave the webhook IDs at their defaults unless you
   have a reason to change them.
2. In lginputhook's web UI (`http://[TV-IP]:1842`), assign the volume keys
   (ID 114 = down, ID 115 = up) these commands:
   ```
   curl -s -X POST http://[HA-IP]:8123/api/webhook/onkyo_volume_down
   curl -s -X POST http://[HA-IP]:8123/api/webhook/onkyo_volume_up
   ```
3. To survive TV reboots, call the lginputhook autostart Luna method once via
   SSH so it registers itself in `/var/lib/webosbrew/init.d/`:
   ```
   ssh root@[TV-IP] "luna-send -n 1 luna://org.webosbrew.inputhook.service/autostart '{}'"
   ```

---

#### Blueprint: Power Sync with TV

`blueprints/automation/onkyo_power_sync.yaml`

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fsonarkon%2FSlop-Assistant%2Fmain%2Fblueprints%2Fautomation%2Fonkyo_power_sync.yaml)

Follows the TV's power state. When the TV turns on: powers the receiver on,
waits a configurable delay, then switches its input to optical. When the TV
has been off for a configurable delay: sends the receiver to standby.

**Requires:** ESP32 with `onkyo-ri-bridge.yaml` + TV as a `media_player`
entity in Home Assistant.

**Setup:**

1. Import the blueprint and create an automation.
2. Select your TV's `media_player` entity, and the power/input button entities
   from the Onkyo RI Bridge.
3. Tune the input-switch delay (default: 3 s) and power-off delay (default:
   2 min) to taste.

---

More tools will land here over time — check back for updates.
