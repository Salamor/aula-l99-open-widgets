# 🖥️ Aula L99 Open Widgets

> Unlocking the full potential of the Aula L99 touchscreen — custom widgets, dynamic displays, and a Stream Deck-like experience, built by the community.

---

## What is this project?

The **Aula L99** is a mechanical keyboard featuring a built-in **3.98" IPS touchscreen (320×480)**. Out of the box, it supports GIF/image uploads, a calculator, a numpad, media controls, and LED management — all through the official AULA driver.

But the screen is capable of so much more.

This project aims to **reverse engineer the USB communication protocol** between the AULA driver and the L99, and build an **open-source software layer** on top of it — enabling fully custom, dynamic widgets on the keyboard's touchscreen.

Think: Spotify now playing, Discord status, system stats, weather, custom macros — all displayed and interactive, directly on your keyboard.

---

## The vision

The L99's touchscreen has the surface area and resolution to act as a **mini Stream Deck built into your keyboard**. No extra device, no extra desk space. Just your keyboard, doing more.

What we want to build:
- 🎵 **Media widget** — current track, artist, album art from Spotify / Apple Music
- 💬 **Discord widget** — voice channel status, mute/deafen toggle, notifications
- 📊 **System stats** — CPU, GPU, RAM usage in real time
- 🌤️ **Weather widget** — local forecast at a glance
- ⚡ **Custom macro buttons** — tap to trigger shortcuts, launch apps, run scripts
- 🚀 **App launcher** — tap to bring any window to the foreground or launch it, on screen 1 or 2
- 🎨 **Fully customizable layouts** — multiple pages, drag, resize, theme your widgets

---

## Why this matters — a real use case

This project started from a very concrete frustration.

Picture a dual-monitor setup: a 32" main display and a 13" secondary screen. At any given time, there are 6–8 apps open — Discord, Spotify, a browser, League of Legends, Blitz.gg, a video tutorial, a work project. Switching between them means Alt+Tab, losing focus, breaking flow.

The obvious solution is a tablet as a secondary touch panel — a Galaxy Tab, an iPad. In practice: lag, unresponsive touch when used as a second screen, and yet another device taking up desk space.

The L99 already sits on the desk. It already has a touchscreen. **Why not use it?**

Here's what the ideal setup looks like in practice:

- 🎮 **While playing League of Legends** — tap Blitz.gg to pull up champion stats on screen 2, tap Discord to bring up a friend's screen share, tap Spotify to skip a track — all without Alt+Tab, all without ever leaving the game
- 💼 **While working** — a custom layout with quick-launch buttons for the exact tools needed for that project, switching profiles between "gaming mode" and "work mode" with a single tap
- 🎵 **Always** — current track visible at a glance, one tap to skip, no need to switch windows

This is the problem a $300 Stream Deck solves — as a separate device, on an already crowded desk. The L99's built-in touchscreen could solve it more elegantly, at no extra cost, with no extra space.

That's what this project is building toward.

---

## Why this is uniquely interesting

### The L99 screen is unlike any other keyboard screen on the market

The keyboard-with-screen market has exploded in 2024–2025, but existing products fall into distinct categories — each with limitations:

| Keyboard | Screen | PC control | Open ecosystem |
|---|---|---|---|
| Ajazz AKP815 | 4.95" grid of 15 fixed LCD buttons | Per-button images via HID | ❌ Partial |
| Ajazz AKP846 | 10.1" extended display (USB monitor) | Full OS-level display | ❌ DisplayLink |
| Ajazz AK650 / AK35I V3 | 0.85"–1.14" TFT info display | GIF upload only, offline after | ❌ Closed |
| **AULA L99** | **3.98" free canvas, fully tactile** | **Pixel-level, real-time via COM** | **🔶 This project** |

The L99 is the only keyboard where the screen accepts a full custom JPEG over a serial port in real time, with no fixed grid, no locked layout — and nobody has documented how to do this without the official driver. Until now.

### The screen protocol has never been publicly documented

Our Wireshark captures are the **first public documentation** of the L99 screen protocol. No GitHub repo, no forum thread, no prior art exists for `VID EEEF / PID 268A` on a COM port for this device. We are starting from zero — and we have already made significant progress.

### The devs themselves said it's possible
AULA's own team confirmed that custom widget development is technically achievable — it's just not something they've exposed through their official software.

---

## Current status

### Hardware identification ✅

| Component | VID | PID | Interface type |
|---|---|---|---|
| Keyboard (keys, RGB, macros) | `0C45` (SONiX) | `800A` | USB HID |
| Screen | `EEEF` | `268A` | USB Serial (COM port) |

The L99 exposes **two separate USB devices** when connected. The keyboard and screen communicate independently. This is a key architectural finding: the screen is not controlled through the keyboard's HID interface — it has its own dedicated serial channel.

### Screen protocol — partially reverse engineered 🔶

Two Wireshark captures have been analyzed (`captures/capture_red_picture.pcapng` and `captures/capture_luffy_Picture.pcapng`). Key findings:

**Image transfer format (confirmed):**
```
[64-byte header]
  Bytes 00–04 : 12 34 56 78 02     ← Magic signature + command type (0x02 = send image)
  Bytes 05–07 : 00 00 00           ← Padding
  Bytes 08–11 : [height, LE 32bit] ← e.g. E0 01 00 00 = 480px
  Bytes 12–15 : [width,  LE 32bit] ← e.g. E0 01 00 00 = 480px
  Bytes 16–55 : 00 × 40            ← Padding
  Bytes 56–59 : 02 00 00 00        ← Unknown (possibly quality/type)
  Bytes 60–63 : [JPEG size, LE]    ← Exact byte count of the following JPEG

[JPEG data]
  ff d8 ff e0 ...                  ← Standard JPEG, no modification
```

**Refresh trigger (confirmed):** After the image transfer, a HID SET_REPORT command is sent to the keyboard device (`0C45:800A`) to trigger the display refresh. The screen does not update until this command is received.

**Magic bytes origin:** The signature `12 34 56 78` does not match any known ecosystem (not Turing Smart Screen `ef 69`, not Ajazz/Mirabox HID). This is an AULA/SONiX proprietary protocol with no prior public documentation.

### What still needs to be captured

| Feature | Status |
|---|---|
| Image push to screen | ✅ Documented |
| Touch input events (tap X/Y coordinates) | ⏳ Not captured yet |
| Screen brightness control | ⏳ Not captured yet |
| Screen on/off command | ⏳ Not captured yet |
| Navigation between pages (official driver UI) | ⏳ Not captured yet |
| Keyboard RGB commands (`0C45:800A` side) | ⏳ Not captured yet |
| Refresh trigger command (exact bytes) | 🔶 Partially identified |

---

## Technical approach

### Phase 1 — Protocol reconnaissance (in progress)
1. ✅ Identify VID/PID of all exposed USB devices
2. ✅ Capture image push traffic (2 captures analyzed)
3. ✅ Decode image transfer format and magic bytes
4. ⏳ Capture touch input events (next priority)
5. ⏳ Capture brightness, on/off, and other screen controls
6. ⏳ Capture keyboard-side HID commands (RGB, macros)

### Phase 2 — Build a minimal Python bridge
7. Send a custom JPEG to the screen using `pyserial`, without the official driver
8. Read touch coordinates from the COM port
9. Trigger the HID refresh command via `hidapi`

### Phase 3 — Widget framework
10. Build a lightweight app that renders widgets as JPEGs and pushes them to the screen in a loop
11. Map touch zones to actions (tap → launch app, skip track, mute Discord, etc.)
12. Integrate with public APIs: Spotify Web API, Discord RPC, OpenWeatherMap, etc.
13. Design a simple config system for widget layout and profiles

---

## How you can help

You don't need to be a developer to contribute. Here's what we need right now, in priority order:

**📸 If you have an L99 and can run Wireshark (highest priority):**
We need captures of specific interactions:
- Tapping different areas of the screen (we need touch event packets)
- Changing screen brightness in the official driver
- Navigating between pages in the official driver menu
- Any other interaction with the screen in the official software

See `docs/capture-guide.md` for step-by-step instructions on how to capture with Wireshark + USBPcap.

**🐍 If you know Python + USB/HID:**
Help write the initial `pyserial` bridge to push images to the screen. The image format is documented — we just need the code.

**🔬 If you have an AULA F75 MAX:**
The F75 MAX uses the same keyboard VID/PID (`0C45:800A`). Do your Wireshark captures show a similar `12 34 56 78` magic bytes sequence for the screen? Sharing your captures could confirm whether the screen protocol is shared across AULA models.

**🎨 If you're good at UI/UX:**
Design the widget layout interface, icon sets, themes.

**📝 If you just have ideas:**
Open an issue and describe the widget you'd want. Community input shapes what gets built first.

---

## Captures available

The following Wireshark captures are available in the `captures/` directory:

| File | What it contains | Status |
|---|---|---|
| `capture_red_picture.pcapng` | Sending a solid red 480×480 JPEG to the screen | ✅ Analyzed |
| `capture_complex_picture.pcapng` | Sending a complex image (320×480) to the screen | ✅ Analyzed |

More captures will be added as the protocol is mapped.

---

## Related projects & inspiration

| Project | Description |
|---|---|
| [F75_Initializer](https://github.com/Simon-Martens/F75_Initializer) | Screen initialization for AULA F75 MAX — **same keyboard VID/PID as L99** (`0C45:800A`) |
| [aula-rgb-controller](https://github.com/veysiemrah/aula-rgb-controller) | Complete RGB controller for AULA F87 TK — 191 commits, full protocol docs in C |
| [Aula-F87-Controller](https://github.com/marcoslor/Aula-F87-Controller) | Protocol documented, Python CLI + WebHID app for AULA F87 |
| [OpenRGB issue #5166](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5166) | Complete wireless protocol reverse engineered for AULA F99 |
| [aegis-2067usb](https://github.com/progzone122/aegis-2067usb-custom-software) | Open-source custom software for AULA F2067 — protocol fully documented |
| [win-68-he-tool](https://github.com/caioalonso/win-68-he-tool) | Local copy of official AULA web driver in readable JavaScript |
| [Rangoli](https://github.com/rnayabed/rangoli) | Open-source RK keyboard software via USB reverse engineering |
| [turing-smart-screen-python](https://github.com/mathoudebine/turing-smart-screen-python) | Python library for USB serial screen devices — similar architecture, different protocol |
| [OpenRazer](https://github.com/openrazer/openrazer) | Community Razer driver for Linux — proof that this model works at scale |
| [Elgato Stream Deck](https://github.com/elgato/streamdeck) | The gold standard for what keyboard-adjacent widgets can look like |

---

## Hardware specs (confirmed)

| Spec | Value |
|---|---|
| Screen | 3.98" IPS touchscreen |
| Resolution | 320 × 480 px |
| Connection | USB-C (wired) |
| Keyboard USB device | `VID 0C45` / `PID 800A` (SONiX chip) |
| Screen USB device | `VID EEEF` / `PID 268A` (COM serial port) |
| Screen protocol | Serial, magic bytes `12 34 56 78`, JPEG payload |
| OS support | Windows only (official driver) |
| Driver download | [soaidriver.com](https://soaidriver.com) |

---

## Community

This project was started by a user who wanted a keyboard that could do more — not a developer. If you're in the same boat, you're in the right place.

- 💬 Discussion: [GitHub Issues](../../issues) — share ideas, ask questions, post captures
- 🧵 Reddit thread: *[link to be added]*

---

## License

MIT — free to use, modify, and distribute.

---

*The Aula L99 has the hardware. Let's build the software.*
