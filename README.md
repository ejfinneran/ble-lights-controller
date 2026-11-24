# Garage Lights

One web page that controls both garage LED lights over Bluetooth LE, from an iPhone (in Bluefy, a free browser on the App Store) or from Chrome on a computer or Android phone. It replaces the two vendor apps and the remote.

| Light | Advertised name | Protocol |
|---|---|---|
| Govee H607C | `Govee_H607C_9D22` | Govee 20-byte packets with an XOR checksum and a keep-alive |
| Generic strip controller (TG201A) | `LED Lights` | ELK-BLEDOM-style 9-byte `7E … EF` packets |

> **Status:** the command bytes come from community reverse-engineering and haven't been confirmed to change the lights yet. Each card has protocol settings so you can find the working format from the phone. See [docs/SPEC.md](docs/SPEC.md).

## Features

- All on / All off, and presets for both lights: **Work light** (white, 100%), **Dim** (warm white, 20%), **Party** (an effect on LED Lights, a color on Govee)
- Per light: connect / reconnect, power toggle, color picker and swatches, brightness slider, status dot
- Up to 3 automatic reconnect attempts if a light drops
- Writes go out at most one every 50 ms, keeping only the newest value, so dragging a control doesn't flood the light
- Colors, brightness and protocol settings are remembered on each device (localStorage)
- A log panel showing every packet sent, in hex

## Running it

The whole app is `index.html`, with no build step and no backend. Web Bluetooth only works on HTTPS or `localhost`.

**On a computer (Chrome or Edge):**

```sh
python3 -m http.server 8000
# open http://localhost:8000
```

**On an iPhone:** open **https://ejfinneran.github.io/ble-lights-controller/** in Bluefy. Every push to `main` redeploys it through GitHub Actions (`.github/workflows/pages.yml`). Safari and the other iOS browsers can't do this, because they have no Web Bluetooth.

The first time, tap **Connect** on each card and choose the light in the picker. After that, **Reconnect** connects without the picker for the rest of that page session.

## Finding the right protocol

If a light connects but doesn't react, open **Protocol settings** on its card, change one setting, and press **Send test**:

| Light | Setting | Options |
|---|---|---|
| Govee | Color command | `02` (default) → `0D` → `15` |
| Govee | Brightness range | 0–255 (default) or 0–100: pick the one where 100% looks fully bright |
| LED Lights | Packet format | spec (`7E 00 …`, default) or alt (`7E <len> …`) |
| LED Lights | Party effect / speed | effect number and speed 0–100 |

Once a setting works, change its default in `defaultSettings()` in `index.html` and update docs/SPEC.md.

## Limits

- Control works only while the page is open and the phone is within Bluetooth range (about 10 m).
- The page can't read the lights' current state, so the controls show the last values sent from this device.
- iOS hides MAC addresses, so lights are matched by advertised name.

## Repo layout

```
index.html     the app (HTML, CSS and JS inline)
docs/SPEC.md   design spec: devices, GATT UUIDs, byte protocols, options considered
AGENTS.md      notes for coding agents and contributors
.github/workflows/pages.yml   deploys index.html to GitHub Pages
```
