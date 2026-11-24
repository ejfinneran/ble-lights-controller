# AGENTS.md

Notes for coding agents and humans working on this repo. Read [docs/SPEC.md](docs/SPEC.md) for the full background.

## What this is

A single static page (`index.html`) that controls two BLE garage lights with the Web Bluetooth API. Its main target is **Bluefy on iOS**, which runs on WebKit. It also runs in Chromium browsers.

## Hard constraints

- **Keep it one file:** inline CSS and JS, no build step, no framework, no npm, no external scripts or fonts. It must work when opened from any static host, offline from the CDN.
- **Use plain browser JS** that runs on Safari / WebKit 15+. Bluefy uses the system WebKit, so don't use APIs that only Chromium has unless you feature-detect them (`getDevices`, `writeValueWithoutResponse` and `watchAdvertisements` are all optional there).
- **Web Bluetooth rules:**
  - `requestDevice()` only works from a user gesture (a tap), so never call it on load or after an `await`.
  - Every service the page uses must be listed in `optionalServices` (`ALL_SERVICES`).
  - It only works on HTTPS or localhost.
- **No backend.** Option B (a laptop bridge in Python) is described in the spec but isn't built. If it is ever built, put it in a separate directory and keep the same driver interface.
- Don't widen the device filters to `acceptAllDevices`. The name-prefix filters keep the picker short, and they keep out the nearby `GVH5075_*` Govee thermometers.

## Code map (inside `index.html`)

| Part | What it does |
|---|---|
| Constants | Service and characteristic UUIDs, write interval (50 ms), color swatches |
| `Light` (base class) | Connect and disconnect, reconnect retries, and the write queue (`send(key, bytes)` keeps only the latest value per key; `tick()` writes one packet per interval) |
| `GoveeLight` | `goveePacket()` pads to 20 bytes and appends the XOR checksum. Sends a keep-alive every 2 s. Color command is `02`, `0D` or `15`; brightness range 255 or 100 |
| `ElkLight` | Two packet tables, `ELK_VARIANTS.spec` and `ELK_VARIANTS.alt`, plus `effect(mode, speed)` |
| UI | `buildCard`, `render`, `setPower`, `setColor` and `setBrightness`, plus `PRESETS` and `applyPreset` |
| Storage | `store.get/set`, a wrapper around localStorage with a `garage:` prefix. Keys: `state:<id>`, `settings:<id>`, `devname:<id>` |

**Driver interface.** Every driver has `connect()`, `power(on)`, `color(r, g, b)`, `brightness(pct)`, `disconnect()` and `party(state)`. A new brand is a new subclass of `Light` that sets `filters`, `serviceUuid`, `writeUuid`, `defaultSettings()` and `settingsSchema()`, plus one entry in the `lights` array.

## Protocol status (important)

The byte formats are **best guesses that haven't been confirmed** against the real lights. That's why the protocol choices are settings rather than constants. When someone confirms which format works:

1. Change that light's `defaultSettings()`.
2. Update the tables and "Next steps" in `docs/SPEC.md`.
3. Don't remove the alternative formats unless asked. They're still useful for other units.

Never present a protocol as working unless someone actually saw the light react.

## Testing

There's no automated test suite, and real verification needs the hardware.

- Desktop: run `python3 -m http.server 8000` and open `http://localhost:8000` in Chrome. Bluetooth has to be on and the lights in range (the garage laptop `hpgarage` can reach both).
- Phone: deploy to the HTTPS host and open it in Bluefy.
- Use the in-page **Log** panel to check the exact bytes sent.
- Without hardware, at least load the page and check the browser console for errors. When `navigator.bluetooth` is missing, the page should show the "no Web Bluetooth" banner and still render.

## Style

- 2-space indent, single quotes, semicolons, `'use strict'`.
- Keep the UI usable at phone width (about 375–430 px) with large tap targets. It's dark-themed by design.
- Prefer small, readable functions over abstraction. This is a home-garage tool, not a framework.

## Privacy note

`docs/SPEC.md` contains the lights' Bluetooth MAC addresses and the laptop's hostname. These are low-risk, but check before making the repo public (GitHub Pages hosting from a free account needs a public repo).
