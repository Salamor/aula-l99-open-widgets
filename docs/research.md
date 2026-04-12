# Research & Prior Art

This document collects all the research gathered before and during the reverse engineering process. It serves as a reference for contributors and a record of why certain technical decisions were made.

---

## 1. AULA as a manufacturer

AULA is a Chinese peripheral brand whose keyboards are built on chips manufactured by **SinoWealth** (also referred to as SUOAI in some documentation). This is a common practice in the budget mechanical keyboard market — multiple brands (AULA, Royal Kludge, Redragon, etc.) buy from the same chip suppliers and ship very similar firmware stacks under different branding.

**Key implication:** reverse engineering work done on one AULA keyboard is very likely to carry over to other AULA models, including the L99.

### Firmware identification
A community analysis on r/aula (by user dorok15) identified the firmware chip as **Sino Wealth SH68F073 (8051 MCU)**. Key findings from that analysis:
- The keyboard uses standard USB HID feature reports — not a locked-down proprietary driver
- RGB, key remapping, macros, profiles are all handled via HID commands
- Key matrix + FN layers + config data were extractable
- Firmware updates use standard ISP + Intel HEX format
- A macOS/Linux replacement app is considered feasible
- Open-source tools already exist for similar SinoWealth chips

---

## 2. USB identifiers — the VID/PID system

Every USB device has a **Vendor ID (VID)** and a **Product ID (PID)**. The VID identifies the manufacturer, the PID identifies the specific product.

From the OpenRGB issue tracker, the AULA F75 and F99 have been identified as:

```
VID: 258A (wired)
PID: 010C
```

**`VID 258A` appears to be shared across the entire AULA keyboard lineup.** This is confirmed across multiple models and community projects. Notably, the Royal Kludge RK61 also uses `VID 258A` (PID `004a` and `007a` depending on variant), confirming that both AULA and RK share the same SinoWealth chip family.

**Important note:** The F75 MAX uses a different VID — `0C45` (SONiX) 
instead of `258A` (SinoWealth). Same physical chip family but different 
USB identifier. To be verified for the L99.

When the L99 arrives, the first step is to verify its VID/PID via Windows Device Manager. If VID is `258A`, the L99 is in the same family.

### How to check VID/PID on Windows
1. Open Device Manager
2. Find the keyboard under "Human Interface Devices"
3. Right-click → Properties → Details → Hardware IDs
4. Look for `VID_XXXX&PID_XXXX`

---

## 3. USB interface structure (typical AULA layout)

Based on research across multiple AULA models (confirmed by umesh70's analysis of the F87 on r/AskReverseEngineering), the keyboard exposes **multiple USB interfaces simultaneously**:

| Interface | Purpose |
|---|---|
| Boot HID | Standard keypress events (works on any OS) |
| Media HID | Volume, play/pause, media shortcuts |
| Vendor Specific (UsagePage 0xff00) | Proprietary communication with the driver software — repeated multiple times |

The **Vendor Specific interface** is where everything interesting happens — RGB control, macro programming, and (on the L99) screen commands. This is the interface to target during reverse engineering.

---

## 4. Packet structure — confirmed findings

### Wired mode (AULA F87 / F75 family)
Based on multiple open-source projects and community captures:
- Commands are sent as **64-byte HID feature reports**
- Checksum: `sum(packet[0:31]) & 0xFF` (confirmed by Simon-Martens for F75 MAX)
- 4-step write protocol (confirmed in veysiemrah's protocol notes)
- Per-key color encoding uses planar RGB format

### Wireless mode (AULA F99 — fully documented by rodrigost23)
Packets have **20 bytes** structured as follows:

| Byte(s) | Size | Group | Description |
|---|---|---|---|
| 0-4 | 5 | Header | Command and sequencing info |
| 5-18 | 14 | Payload | The actual data |
| 19 | 1 | CRC | Sum of bytes 0-18 & 0xFF |

Header breakdown:
| Byte | Field | Values | Description |
|---|---|---|---|
| 0 | Signature | 0x13 | Static, part of command signature |
| 1 | Command type | 0x05, 0x07, **0x88** | **0x88 = send LED color info** |
| 2 | Total packets | 0x01-0x0N | Total packets in message |
| 3 | Packet index | 0x00-0x0N-1 | Current packet index |
| 4 | Mode + Data length | 0x10 or 0x20 + N | 0x10 = per-key, 0x20 = solid color |

Example — set full keyboard red:
`0x13, 0x88, 0x1, 0x0, 0x23, 0xff, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0xbe`

Model IDs embedded in protocol: `0xa4` = F99, `0xcd` = F75. **These IDs are shared between wireless and wired protocols**, meaning SignalRGB's Sinowealth plugin (which supports wired mode) uses the same IDs.

### Screen-specific hypothesis
For image/screen data, the protocol likely uses "tunneling" — the same basic packet structure as RGB commands, but with much denser payloads to transfer pixel data. Simon-Martens confirmed that the F75 MAX screen uses 64-byte HID feature reports on interface 3, with a 4-packet initialization sequence.

---

## 5. Existing open-source projects

### AULA-specific — Rank S (most relevant)

| Project | Model | What's done | Language | Link |
|---|---|---|---|---|
| aula-rgb-controller | AULA F87 TK | Full RGB controller, 191 commits, complete protocol documentation in `tools/protocol_notes.md`, GTK4 GUI + daemon + CLI | C | [GitHub](https://github.com/veysiemrah/aula-rgb-controller) |
| F75_Initializer | AULA F75 MAX | **Controls the screen from Linux**, RTC sync via HID, captures included (`keyboard5/6/7.pcapng`), documents 4-packet init sequence | Python | [GitHub](https://github.com/Simon-Martens/F75_Initializer) |
| win-68-he-tool | AULA WIN60/68 HE | **Copy of official AULA web driver JS source** — inspectable in browser DevTools, reveals packet format directly | JavaScript | [GitHub](https://github.com/caioalonso/win-68-he-tool) |
| OpenRGB issue #5166 | AULA F99 | **Full wireless protocol reverse engineered** by rodrigost23, captures + byte-level documentation, working PoC in OpenRGB PRs !3026 and !3027 | — | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5166) |
| NollieL/SignalRgb_CN_Key | AULA Series (F75, F99, etc.) | Full AULA protocol in readable JavaScript — `Aula Series.js` contains the complete communication logic for SignalRGB | JavaScript | [GitHub](https://github.com/NollieL/SignalRgb_CN_Key) |

### AULA-specific — Rank A (solid reference)

| Project | Model | What's done | Language | Link |
|---|---|---|---|---|
| Aula-F87-Controller | AULA F87 | Protocol documented, Python CLI + WebHID app, captures in `/captures`, `docs/PROTOCOL.md` | Python / TypeScript | [GitHub](https://github.com/marcoslor/Aula-F87-Controller) |
| aula-f87pro | AULA F87 Pro | CLI Python, RGB control on Linux, udev rules, VID `258A:010C` confirmed | Python | [GitHub](https://github.com/Ahorts/aula-f87pro) |
| aula-keyboard | AULA (generic) | Webapp React + TypeScript in progress, WebHID approach | TypeScript | [GitHub](https://github.com/KayviaHarriott/aula-keyboard) |
| aula_contol-f87 | AULA F87 | Jupyter Notebook exploration + C test code, active analysis of vendor-specific HID interfaces | Python / C | [GitHub](https://github.com/umesh70/aula_contol-f87) |
| tarantula | AULA (generic) | Open-source Rust library for AULA keyboard configuration, early stage | Rust | [GitHub](https://github.com/sangwon090/tarantula) |
| aula-f75-configurator | AULA F75 | Cross-platform WebHID configurator in progress, updated recently | TypeScript | [GitHub](https://github.com/AmoabaKelvin/aula-f75-configurator) |
| aegis-2067usb | AULA F2067 | Full USB protocol reverse engineered, CLI tool | Rust | [GitHub](https://github.com/progzone122/aegis-2067usb-custom-software) |


### AULA-specific — OpenRGB tracker

| Issue | Model | Status | Link |
|---|---|---|---|
| #5166 | AULA F99 | Full protocol documented by rodrigost23, PRs submitted | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5166) |
| #4232 | AULA F75 | VID/PID identified | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/4232) |
| #5253 | AULA F108Pro | Device request opened | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/5253) |

### Royal Kludge (same VID family, similar protocol)

| Project | What's done | Link |
|---|---|---|
| Rangoli | Full USB protocol reverse engineered via Wireshark, cross-platform app | [GitHub](https://github.com/rnayabed/rangoli) |
| Kludge Knight | Browser-based WebHID configurator built on Rangoli's protocol research | [GitHub](https://github.com/vinc3m1/kludgeknight) |
| rkcu | RK61 CLI Python, 22 stars, confirms VID `258A:004a` | [GitHub](https://github.com/oddlyspaced/rkcu) |
| rk918_rk919_re | RK918/919 firmware extraction, MCU identification (SN32F248B), QMK port in progress | — | [GitHub](https://github.com/cvbicwh/rk918_rk919_re) |

**Key observation:** The official AULA driver and the official Royal Kludge driver are visually nearly identical. Both share VID `258A` (SinoWealth). The protocol structures documented in Rangoli are very likely applicable — at least partially — to the L99.

### Other relevant ecosystems

**SignalRGB** has a working Sinowealth plugin that supports AULA keyboards in wired mode, using the same model IDs as the wireless protocol documented by rodrigost23. Worth examining as an additional reference.

---

## 6. The screen — what makes the L99 unique

Most AULA keyboards (F75, F87, F2067) have no screen. The L99 adds a **3.98" IPS touchscreen at 320×480 resolution**, which is the entire point of this project.

The closest existing work on AULA screens is **Simon-Martens/F75_Initializer**, which targets the F75 MAX's smaller display and has already mapped the initialization protocol. This is the most directly applicable prior art for the L99 screen work.

Other comparable keyboards with screens in the market:
- **Royal Kludge S98 / M87 / M75** — small OLED/TFT info displays (not touch, not interactive)
- **MiraBox K1W** (Kickstarter, 2026) — 6 individual LCD keys, browser-based Web UI, open SDK promised
- **MMOBIEL stream deck keyboard** — 6 LCD macro keys, uses Elgato Stream Deck software
- **Corsair Galleon 100SD** — integrated Stream Deck keys, ~350€

**None of these combine a large touchscreen with an open software ecosystem.** The L99 has the hardware. This project aims to build the software.

---

## 7. Why the screen protocol is likely discoverable

The screen almost certainly communicates via the same **Vendor Specific USB interface** used for RGB and macros — just with larger payloads. Simon-Martens confirmed this approach for the F75 MAX: interface 3, 64-byte feature reports, 4-packet init sequence.

Wireshark + USBPcap will capture every packet exchanged during driver operations. Once captured, comparing them against documented protocols (marcoslor's `docs/PROTOCOL.md`, veysiemrah's `tools/protocol_notes.md`, rodrigost23's F99 documentation) should reveal the pattern quickly.

---

## 8. AULA WebHID drivers — an inspectable entry point

AULA has developed browser-based WebHID drivers for some of their keyboard lines:

| Model | URL |
|---|---|
| WIN60/68HE Standard | [device.aulacn.com](https://device.aulacn.com) |
| WIN60/68HE PRO/MAX | [win.aulacn.com](https://win.aulacn.com) |
| HERO68 Standard | [hero.aulastar.com](https://hero.aulastar.com) |
| HERO68 ULTRA / PRO/MAX | [magnet.aulastar.com](https://magnet.aulastar.com) |

These are for Hall Effect keyboard lines, not the L99. But their JavaScript source is fully readable in browser DevTools. **caioalonso archived a local copy** of the WIN60/68HE web driver at [github.com/caioalonso/win-68-he-tool](https://github.com/caioalonso/win-68-he-tool) — this is the most accessible entry point to understand how AULA structures HID communication without any hardware.

---

## 9. Community discussions

### r/aula — dorok15
Analysis post confirming:
- Standard USB HID feature reports (not locked-down)
- Sino Wealth chipset confirmed (same as RK)
- Firmware: **SH68F073 (8051 MCU)**
- RGB/macros/profiles all via HID commands
- macOS/Linux replacement app considered feasible
- No working replacement tool yet — author looking for HID/USB contributors

### r/AskReverseEngineering — umesh70
Post from the same person behind [aula_contol-f87](https://github.com/umesh70/aula_contol-f87):
- Identified Interface 1 with UsagePage 0xff00 (vendor specific) as the RGB control channel
- Has HID descriptor dump available to share
- Plans to capture USB traffic with Wireshark + USBPcap in a VM
- Actively looking for help decoding report format
- 0 replies so far — **high priority contact**

---

## 10. People to potentially contact (priority order)

| Priority | Who | Why | Where |
|---|---|---|---|
| 🥇 1 | rodrigost23 | Reverse-engineered the complete F99 wireless protocol, submitted working OpenRGB PRs, active in AULA ecosystem | [GitLab](https://gitlab.com/rodrigost23) |
| 🥈 2 | veysiemrah | Built the most complete AULA open-source project (191 commits), full protocol docs, same VID | [GitHub](https://github.com/veysiemrah/aula-rgb-controller) |
| 🥈 2 | Simon-Martens | **Already working on AULA screen control from Linux**, has captures and working init code | [GitHub](https://github.com/Simon-Martens/F75_Initializer) |
| 🥉 3 | dorok15 | Identified SH68F073 MCU, confirmed standard HID feature reports | r/aula |
| 🥉 3 | darkdex52 | F75 MAX user with Wireshark captures available, opened OpenRGB issue #5326 | [GitLab](https://gitlab.com/darkdex52) |
| 🥉 3 | umesh70 | Active reverse engineering of F87, looking for collaborators, same frustration | [GitHub](https://github.com/umesh70/aula_contol-f87) |
| 🥉 3 | marcoslor | Built F87 controller, protocol documented, Python + WebHID | [GitHub](https://github.com/marcoslor/Aula-F87-Controller) |
| 4 | rnayabed | Built Rangoli (RK reverse engineering), similar approach | [GitHub](https://github.com/rnayabed/rangoli) |
| 4 | progzone122 | Built aegis-2067usb for AULA F2067 | [GitHub](https://github.com/progzone122/aegis-2067usb-custom-software) |
| 4 | OpenRGB community | Already tracking multiple AULA devices | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB) |

---

*This document will be updated as the project progresses.*
