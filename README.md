# ESPHome Controller for Two-Wire Fairy Lights

Replace the cheap "8 function" controller box on a two-wire LED fairy string with an ESP32 running [ESPHome](https://esphome.io). You get local Home Assistant control, brightness, nine effects, and a live **Speed** and **Fade Depth** slider, with no cloud and no reflashing to tweak anything.

The headline effect is a true **sine-wave crossfade**: the two halves of the string fade in and out as exact inverse sine waves, which looks much smoother than the stock box.

You need three small boards (a microcontroller, an H-bridge and a 5V regulator) plus a capacitor. You keep the string and its original power supply.

---

## Contents

1. [How two-wire fairy lights work](#how-two-wire-fairy-lights-work)
2. [Check your string first](#check-your-string-first)
3. [Parts](#parts)
4. [Wiring](#wiring)
5. [Flashing the firmware](#flashing-the-firmware)
6. [Using it in Home Assistant](#using-it-in-home-assistant)
7. [Effects](#effects)
8. [Migrating from WLED (over the air)](#migrating-from-wled-over-the-air)
9. [Troubleshooting](#troubleshooting)
10. [Further reading](#further-reading)

---

## How two-wire fairy lights work

These strings have only two wires, but they show two separate "sets" of LEDs. The LEDs are wired **anti-parallel**: every other LED faces the opposite way.

- Current flowing one way lights **set A**.
- Current flowing the other way lights **set B**.
- Flip the direction fast enough and both sets look lit at once.

The stock controller does this with an H-bridge, which is just a circuit that can push current through a load in either direction. This project does the same with a DRV8871 H-bridge driven by an ESP32.

ESPHome has a built-in [`hbridge` light platform](https://esphome.io/components/light/hbridge/) for exactly this. It shows up in Home Assistant as a light with a colour-temperature slider: the cool end is set A, the warm end is set B, and the middle is both.

## Check your string first

Before buying parts, check two things.

**1. Is it a two-wire string with a DC power supply?** Look for a wall adapter feeding a small controller box with a mode button, and two wires going out to the lights. If the controller plugs straight into mains with no adapter, **stop**: this guide doesn't apply and that string isn't safe to work on.

**2. What voltage is the supply?** Read the adapter label, or measure its output with a multimeter. The string this was built for runs on a **29V** supply; 24-36V is common for long mains-adapter strings.

| Supply voltage | H-bridge to use |
| --- | --- |
| 6.5-45V (most plug-in strings) | **DRV8871**, as in this guide |
| Under 6.5V (battery or USB strings) | DRV8871 won't start. Use a DRV8833 (2.7-10.8V) and power the ESP32 from USB instead of a buck regulator |

The common hobby drivers (DRV8833, TB6612, L9110S) are only rated to about 10-15V. On a ~30V string they will die, which is why the DRV8871 is the default here.

## Parts

| Part | Why this one |
| --- | --- |
| **Seeed XIAO ESP32-C3** | Tiny, cheap, runs ESPHome. **Attach the u.FL antenna it comes with**: Wi-Fi is poor without it. |
| **DRV8871 H-bridge breakout** (e.g. Adafruit #3190) | Rated 6.5-45V, so it handles 30V strings with headroom. Takes two PWM inputs and needs no extra transistors. |
| **Pololu D36V6F5** step-down regulator | Fixed 5V output, accepts up to 50V input. Generic "5-30V" buck modules are too close to a 29V supply to be trusted. |
| **10-47µF electrolytic capacitor, 50V or higher** | Goes across the supply input to absorb the spike when the adapter is plugged in. |
| The string's **original power supply** | Reused as-is. |
| Hookup wire, a small enclosure, heat-shrink | |

## Wiring

```
                 +-------------------+
 Supply + ──┬────┤ VM                |
            │    |     DRV8871   OUT1├──── string wire 1
            │    |               OUT2├──── string wire 2
 Supply − ──┼─┬──┤ GND               |
            │ │  |  IN1    IN2       |
            │ │  +---┬──────┬--------+
            │ │      │      │
            │ │      │      │      +-------------------+
            │ │      │      └──────┤ D3 (GPIO5)        |
            │ │      └─────────────┤ D2 (GPIO4)        |
            │ │                    |   XIAO ESP32-C3   |
            │ │  +-----------+     |                   |
            ├─┼──┤ VIN       |     |                   |
            │ ├──┤ GND  VOUT ├─────┤ 5V                |
            │ │  +--┬--------+     |                   |
            │ │     └──────────────┤ GND               |
            │ │   D36V6F5 (5V)     +-------------------+
            │ │
           [ capacitor ]  (+ to Supply +, − to Supply −)
```

Connection list:

| From | To |
| --- | --- |
| Supply + | DRV8871 **VM**, D36V6F5 **VIN**, capacitor **+** |
| Supply − | DRV8871 **GND**, D36V6F5 **GND**, capacitor **−** |
| D36V6F5 **VOUT** | XIAO **5V** |
| D36V6F5 **GND** | XIAO **GND** (all grounds must be common) |
| XIAO **D2** (GPIO4) | DRV8871 **IN1** |
| XIAO **D3** (GPIO5) | DRV8871 **IN2** |
| DRV8871 **OUT1 / OUT2** | The two string wires. Polarity doesn't matter; swapping them just swaps which set is "A". |

> [!WARNING]
> **Never plug USB into the XIAO while the high-voltage supply is connected.** The XIAO has no diode between its USB 5V and its 5V pin, so the buck regulator and your computer's USB port would fight each other. After the first flash, update over Wi-Fi.

Avoid XIAO pins D0, D8 and D9 (GPIO2/8/9). They are boot-strapping pins and can stop the board booting if something pulls on them.

## Flashing the firmware

### 1. Add your secrets

The config keeps every password in ESPHome's `secrets.yaml`. In the ESPHome Builder (Home Assistant add-on or standalone dashboard), open **Secrets** in the top-right menu and add the keys from [`secrets.example.yaml`](secrets.example.yaml):

```yaml
wifi_ssid: "YourNetwork"
wifi_password: "YourWiFiPassword"
api_encryption_key: "..."   # 32 random bytes, base64
ota_password: "..."
fallback_password: "..."    # at least 8 characters
```

To generate an API encryption key, run `openssl rand -base64 32`, or copy the one ESPHome Builder puts in a freshly created device config.

### 2. Add the config

Create a new device in ESPHome Builder, then replace its YAML with [`fairy-lights.yaml`](fairy-lights.yaml). To rename it, edit the `substitutions` at the top:

```yaml
substitutions:
  device_name: fairy-lights     # hostname; keep it <= 23 chars
  friendly_name: Fairy Lights
  pin_a: GPIO4                  # XIAO D2
  pin_b: GPIO5                  # XIAO D3
```

Keep `device_name` to 23 characters or fewer. The fallback access point is named `<device_name>-fallback`, and Wi-Fi network names max out at 32 characters.

### 3. First flash over USB

With the **lights' power supply unplugged**, connect the XIAO to your computer by USB and choose **Install → Plug into this computer** (or use [web.esphome.io](https://web.esphome.io)). If the board isn't detected, hold the **BOOT** button while plugging it in.

### 4. Every flash after that: Wi-Fi

Unplug USB, connect the lights' supply, and the device joins your Wi-Fi. From then on use **Install → Wirelessly**.

If it can't reach your Wi-Fi, it starts an access point called `<device_name>-fallback`. Join it with your `fallback_password` and a captive portal lets you enter Wi-Fi details.

## Using it in Home Assistant

Home Assistant discovers the device automatically. Go to **Settings → Devices & services**, accept the new ESPHome device, and you get:

| Entity | What it does |
| --- | --- |
| `light.fairy_lights` | On/off, brightness, colour-temp slider (cool = set A, warm = set B, middle = both), effect list |
| `number.fairy_lights_speed` | Multiplier on every effect's timing, 0.1-5x. 2x = twice as fast. |
| `number.fairy_lights_fade_depth` | 0-100%. At 100% fades go fully dark; lower values give a gentle ripple that never goes out. |
| `sensor.fairy_lights_wifi_signal`, `binary_sensor.fairy_lights_status` | Diagnostics |

Home Assistant may put the device's area name in front of the number entity IDs, e.g. `number.living_room_fairy_lights_speed`.

Speed and Fade Depth apply live while an effect runs, without restarting it, and are remembered across reboots. The light itself always powers up **off** (`RESTORE_DEFAULT_OFF`), so a power cut won't leave it on.

Example script to switch to the crossfade:

```yaml
action: light.turn_on
target:
  entity_id: light.fairy_lights
data:
  brightness_pct: 100
  effect: Crossfade
```

## Effects

Times are at Speed 1.0x.

| Effect | What it does |
| --- | --- |
| **Alternate** | A and B swap with a hard cut, 0.5s each |
| **Blink** | Whole string on/off, 0.5s each |
| **Sequential** | A → both → B → off, 0.4s per step (like the stock controller) |
| **Double Flash** | A flashes twice, then B flashes twice, 1s per set |
| **Crossfade** | A and B as exact inverse sine waves, 1s per fade |
| **Breathe** | Both sets fade in and out together, 2s per fade |
| **Waves** | Sine waves a quarter-cycle apart, so the sets overlap as they rise and fall (stock "In Waves"), 2s per fade |
| **Slow Glow** | One set at a time glows up and back down, never overlapping (stock "Slo-Glo"), 1.5s per fade |
| **Twinkle** | Each set drifts smoothly to a new random brightness, 0.8s per drift |

Each effect tracks its own phase from elapsed time × Speed, so changing Speed mid-effect never makes it jump. The fades go through ESPHome's gamma correction, so *perceived* brightness follows the sine curve, not just the raw PWM value.

The PWM runs at about 19.5kHz, which is above hearing range, so there's no whine from the LEDs or wiring.

To add your own effect, copy one of the `lambda` blocks. `set_cold_white()` is set A and `set_warm_white()` is set B, each 0.0-1.0.

### Why not WLED?

WLED can drive these strings (bus type "PWM CCT" on two pins), but its effects can't animate the A/B balance. Playlist transitions between the two are linear, so a "crossfade" comes out as a triangle wave rather than a smooth sine. ESPHome lambdas give full control over the waveform.

## Migrating from WLED (over the air)

If your controller is already running WLED, you can switch it to ESPHome without opening it up or plugging in USB. This was tested going from WLED 16.

1. In ESPHome Builder, choose **Install → Manual download → OTA format**. You get a file like `fairy-lights.ota.bin`.
2. Upload it to WLED's update endpoint. **The `skipValidation` field must come before the file**, or WLED rejects a non-WLED image with "missing compatibility metadata":

   ```sh
   curl -F skipValidation=1 -F update=@fairy-lights.ota.bin http://<wled-ip>/update
   ```

3. ESPHome boots within a few seconds on the same Wi-Fi.
4. In Home Assistant, **delete the old WLED integration entry first**, then accept the ESPHome discovery. Otherwise the new entities get a `_2` suffix.

Save a backup of your WLED config and presets (WLED's **Config → Security & Updates → Backup**) first, in case you want to go back.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Only one set lights, even in the middle of the colour-temp slider | One of IN1/IN2 isn't connected, or is on the wrong GPIO |
| Nothing lights, but the device is online | No common ground between buck, XIAO and DRV8871; or supply below the DRV8871's 6.5V minimum |
| Flaky or dropping Wi-Fi | u.FL antenna not attached |
| Board won't boot after wiring | Something connected to a strapping pin (D0, D8, D9) |
| Audible whine | PWM frequency lowered from 19531Hz; put it back |

## Further reading

- [ESPHome H-bridge light documentation](https://esphome.io/components/light/hbridge/)
- [Home Assistant community: H-Bridge 2-wire fairy light controller](https://community.home-assistant.io/t/h-bridge-2-wire-fairy-light-controller/633484)
- [AusChristmasLighting wiki: Controlling 2-wire multi-function LED strings](https://auschristmaslighting.com/wiki/Controlling-2-wire-Multi-Function-LED-Strings)
- [TI DRV8871 datasheet](https://www.ti.com/product/DRV8871)

## License

[MIT](LICENSE). Build it, change it, share it.
