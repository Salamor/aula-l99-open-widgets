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

## Why it's feasible

### The software similarity clue
The official AULA driver looks **nearly identical** to the Royal Kludge (RK) driver software. This strongly suggests both brands share the same underlying USB controller chip and firmware stack — a common practice among Chinese keyboard manufacturers using SinoWealth chipsets.

### Prior art: Rangoli & Kludge Knight
The Royal Kludge protocol has already been reverse engineered:

- [**Rangoli**](https://github.com/rnayabed/rangoli) — open-source, cross-platform RK software, built by reverse engineering USB packets with Wireshark
- [**Kludge Knight**](https://github.com/vinc3m1/kludgeknight) — browser-based RK configurator, built on top of Rangoli's protocol research

If the L99 shares the same protocol (or a close variant), a significant portion of the reverse engineering groundwork is already done. The **screen-specific commands** would be the new territory to map.

### The devs themselves said it's possible
AULA's own team confirmed that custom widget development is technically achievable — it's just not something they've exposed through their official software.

---

## Current status

| Feature | Status |
|---|---|
| USB protocol analysis | ⏳ Not started |
| Basic communication (RK protocol compatibility test) | ⏳ Not started |
| Screen image push | ⏳ Not started |
| Dynamic widget framework | ⏳ Not started |
| Touch input reverse engineering | ⏳ Not started |

---

## Technical approach

### Phase 1 — Protocol reconnaissance
1. Capture USB traffic between the official AULA driver and the L99 using **Wireshark + USBPcap**
2. Compare captured packets to the documented [RK protocol](https://github.com/progzone122/aegis-2067usb-protocol)
3. Identify which commands control the screen (image push, touch events, etc.)

### Phase 2 — Build a minimal Python bridge
4. Reproduce screen commands using `hidapi` or `pyusb`
5. Push a static image to the screen without the official driver
6. Map touch input events back from the keyboard to the host

### Phase 3 — Widget framework
7. Build a lightweight app that renders widgets as frames and pushes them to the screen in a loop
8. Integrate with public APIs: Spotify Web API, Discord RPC, OpenWeatherMap, etc.
9. Design a simple config system so users can choose and arrange widgets

---

## How you can help

You don't need to be a developer to contribute. Here's what we need:

**🔬 If you have a L99 and can run Wireshark:**
Capture USB traffic during image upload and share the `.pcapng` file. This is the single most important thing that can unblock the project.

**🐍 If you know Python + USB/HID:**
Help analyze packets and write the initial communication layer.

**🎨 If you're good at UI/UX:**
Design the widget layout interface, icon sets, themes.

**📝 If you just have ideas:**
Open an issue and describe the widget you'd want. Community input shapes what gets built first.

---

## Related projects & inspiration

| Project | Description |
|---|---|
| [Rangoli](https://github.com/rnayabed/rangoli) | Open-source RK keyboard software via USB reverse engineering |
| [Kludge Knight](https://github.com/vinc3m1/kludgeknight) | Browser-based RK configurator built on Rangoli's protocol research |
| [aegis-2067usb](https://github.com/progzone122/aegis-2067usb-custom-software) | Open-source custom software for AULA F2067 — protocol fully documented |
| [aula-rgb-controller](https://github.com/veysiemrah/aula-rgb-controller) | Complete RGB controller for AULA F87 TK — 191 commits, full protocol documentation in C |
| [F75_Initializer](https://github.com/Simon-Martens/F75_Initializer) | Screen initialization for AULA F75 MAX from Linux — closest existing work to L99 screen control |
| [Aula-F87-Controller](https://github.com/marcoslor/Aula-F87-Controller) | Protocol documented, Python CLI + WebHID app for AULA F87 |
| [OpenRGB issue #5166](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5166) | Complete wireless protocol reverse engineered for AULA F99 — packet structure, CRC, working PoC |
| [win-68-he-tool](https://github.com/caioalonso/win-68-he-tool) | Local copy of official AULA web driver in readable JavaScript |
| [OpenRazer](https://github.com/openrazer/openrazer) | Community Razer driver for Linux — proof that this model works at scale |
| [Elgato Stream Deck](https://github.com/elgato/streamdeck) | The gold standard for what keyboard-adjacent widgets can look like |

---

## Hardware specs (for reference)

| Spec | Value |
|---|---|
| Screen | 3.98" IPS touchscreen |
| Resolution | 320 × 480 px |
| Connection | USB-C (wired, required for driver) |
| OS support | Windows only (official driver) |
| Driver download | [soaidriver.com](https://soaidriver.com) |

---

## Community

This project was started by a user who wanted a keyboard that could do more — not a developer. If you're in the same boat, you're in the right place.

- 💬 Discussion: [GitHub Issues](../../issues) — share ideas, ask questions, post captures
- 🧵 Reddit thread: *[link to be added after posting on r/MechanicalKeyboards]*

---

## License

MIT — free to use, modify, and distribute.

---

*The Aula L99 has the hardware. Let's build the software.*
