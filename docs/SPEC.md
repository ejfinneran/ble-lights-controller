# Garage Lights Controller: Spec

Nov 17, 2026 · @E.J.

## Summary and status

Goal: control both garage LED lights from an iPhone with one simple page, replacing the two proprietary apps and the remote. Both lights accept Bluetooth LE connections and writes with no pairing or PIN, so a web or native controller is feasible.

- **Verified (2026-11-17):** both lights were found in a scan from the garage laptop (`hpgarage`), both connect, and both GATT tables were read.
- **Verified:** test writes (power on + a color) to both lights completed with no errors.
- **Not yet confirmed:** whether the lights actually reacted (Govee should have turned blue, LED Lights red). The protocols below are best guesses until this is checked.
- **Decided:** Option A (Web Bluetooth page in Bluefy). Version 1 is `index.html` in this repo.

## Devices found

Two lights showed up in the scan from the laptop. The two `GVH5075_*` devices nearby are Govee thermometers, not lights.

| Light | Advertised name | MAC (as seen from Linux) | Model hint | Service | Write char | Notify char |
|---|---|---|---|---|---|---|
| Govee | `Govee_H607C_9D22` | `C5:B5:64:51:9D:22` (random) | H607C | `00010203-0405-0607-0809-0a0b0c0d1910` | `…0a0b0c0d2b11` | `…0a0b0c0d2b10` |
| Generic strip controller | `LED Lights` | `41:42:C4:4C:AC:90` (public) | TG201A in manufacturer data (company ID 0xFFFF), advertises UUID 0x3519 | `0000fff0-…` | `0000fff3-…` | `0000fff4-…` |

The signal from LED Lights was about −71 dBm from the laptop. iOS never exposes MAC addresses: on an iPhone each device gets a per-phone UUID, so the app must match lights by advertised name and service UUID instead.

## BLE protocols

Both lights match protocols the community has already reverse-engineered. The command bytes below are from memory of those projects (approximate, not sourced). Only the rows marked *sent* were written to the lights, and the visible result is still unconfirmed.

### Govee (H607C)

Every command is 20 bytes, written to `…2b11`: byte 0 = `0x33`, byte 1 = command, then arguments, zero-padded, and byte 19 = XOR of bytes 0 to 18. Govee lights usually drop an idle connection, so the app sends a keep-alive (`AA 01`, padded and checksummed) about every 2 seconds while connected.

| Action | Bytes before padding | Notes |
|---|---|---|
| Power on | `33 01 01` | sent |
| Power off | `33 01 00` | |
| Brightness | `33 04 VV` | VV is 0–255 on some models and 0–100 on others; test which |
| Color (older models) | `33 05 02 RR GG BB` | sent (blue) |
| Color (variant) | `33 05 0D RR GG BB` | try if 02 does nothing |
| Color (segmented models) | `33 05 15 01 RR GG BB 00 00 00 00 00 FF 7F` | try if both of the above fail |

### LED Lights (TG201A, ELK-BLEDOM style)

Every command is 9 bytes, `7E … EF`, written to `fff3`, with no checksum and no keep-alive.

| Action | Bytes | Notes |
|---|---|---|
| Power on | `7E 00 04 F0 00 01 FF 00 EF` | sent |
| Power off | `7E 00 04 00 00 00 FF 00 EF` | |
| Color | `7E 00 05 03 RR GG BB 00 EF` | sent (red) |
| Brightness | `7E 00 01 VV 00 00 00 00 EF` | VV = 0–100 |
| Built-in effect | `7E 00 03 MM 03 00 00 00 EF` | MM = effect number |
| Effect speed | `7E 00 02 SS 00 00 00 00 EF` | SS = 0–100 |

Many ELK-BLEDOM projects use a length byte in position 1 instead of `00` (for example color `7E 07 05 03 RR GG BB 10 EF`, power on `7E 04 04 F0 00 01 FF 00 EF`). The app offers this as the "alt" packet format.

If this light ignores both formats, the fallback is to capture what the official app sends: turn on the Bluetooth HCI snoop log on an Android phone, or use the laptop as a sniffer.

## Option A: Web Bluetooth page (chosen)

A single static HTML page talks to both lights directly from the phone using the Web Bluetooth API. On iPhone it runs in Bluefy, a free iOS browser with Web Bluetooth. Safari and every other iOS browser lack Web Bluetooth because Apple makes them all use Safari's engine. The same page runs unchanged in Chrome on Android, macOS, Windows and Linux.

### Architecture

- One `index.html` with inline JS and CSS, and no backend or build step.
- A small driver per brand, each with the same interface: `connect()`, `power(on)`, `color(r, g, b)`, `brightness(pct)`, `disconnect()`.
- `requestDevice` filters on name prefixes (`Govee_`, `LED Lights`) and lists both service UUIDs in `optionalServices`.
- The Govee driver runs the 2-second keep-alive timer and adds the XOR checksum.
- Writes are queued, about one every 50 ms, so dragging the color picker doesn't flood the light. Only the latest pending value is kept.
- Last colors and settings are saved in localStorage.

### Requirements

- HTTPS hosting, e.g. GitHub Pages, Netlify or Cloudflare Pages (all free), or localhost while developing on a desktop.
- Bluefy installed on the iPhone.

### Limits

- A tap is needed to connect: the browser shows a device picker the first time for each light, and the page can't connect silently on load.
- Control works only while the page is open. No background control, widgets or Siri.
- Bluetooth range from the phone (about 10 m, less through walls).
- Bluefy is a third-party app, so it could change or disappear.

## Option B: Laptop bridge + web page (not built)

The garage laptop keeps Bluetooth connections to both lights open and serves a web page over Wi-Fi. Any phone browser can use it, plain Safari included, and it can be saved to the home screen.

- Python service using bleak for BLE and aiohttp for HTTP, run as a systemd service so it starts at boot.
- The same two drivers as Option A, ported to Python. Connections stay open and reconnect automatically if a light drops.
- A small JSON API, called by the same UI as Option A:
  - `GET /api/lights`: list lights with connected state and last known color
  - `POST /api/lights/{id}/power` `{"on": true}`
  - `POST /api/lights/{id}/color` `{"r":255,"g":0,"b":0}`
  - `POST /api/lights/{id}/brightness` `{"pct": 80}`
  - `POST /api/all/power` `{"on": false}`
- Reached at `http://hpgarage.local:8080` on the home network.
- Needs `sudo apt install python3-venv`, then bleak and aiohttp in a venv (pip is not installed on the laptop). The laptop must stay on and in range, with suspend on lid close turned off.
- No login on the API: anyone on the Wi-Fi can control the lights. Add a shared token if guests use it. Reaching it from outside the house needs a VPN such as Tailscale; don't expose the port to the internet.

Both options use the same drivers and the same UI. Only where the Bluetooth radio lives changes, so the drivers carry over if we later switch to B.

## Feature scope (version 1)

One screen with a card per light and an "All" row on top.

- All on / All off buttons
- Per light: on/off toggle, color picker, brightness slider, connection status dot
- Presets that apply to both lights at once: "Work light" (full white), "Dim" (warm white, 20%), "Party" (built-in effect on LED Lights, a color on Govee)
- Connect / reconnect button per light

Out of scope for version 1: schedules, music sync, Govee scenes and per-segment color, and reading the current state back from the lights.

## Next steps and open questions

- [ ] Confirm the lights react. If Govee doesn't change color, try the `0D` and `15` color variants; if LED Lights doesn't, try the alt packet format.
- [ ] Find the Govee brightness range (0–100 or 0–255).
- [ ] Confirm the LED Lights effect numbers (Party defaults to `0x88`).
- [ ] Install Bluefy on the iPhone and pick a static host.
- [ ] Test both lights from the phone.

## Laptop test tools

These were saved on the garage laptop at `/home/ej/garage-lights/tools/` (not in this repo):

```sh
# Govee: build a checksummed packet, then write it
./tools/govee_pkt.py 330100          # power off
./tools/blewrite.sh C5:B5:64:51:9D:22 00010203-0405-0607-0809-0a0b0c0d2b11 $(./tools/govee_pkt.py 330100)

# LED Lights: raw 9-byte packets, several in one connection
./tools/blewrite.sh 41:42:C4:4C:AC:90 0000fff3-0000-1000-8000-00805f9b34fb 7e0004f00001ff00ef 7e00050300ff0000ef
```

`blewrite.sh` scans for 5 seconds, connects, writes each packet one second apart, then disconnects. It only needs `bluetoothctl`.
