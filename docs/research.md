# Research & Prior Art

This document collects all the research gathered before and during the reverse engineering process. It serves as a reference for contributors and a record of why certain technical decisions were made.

---

## 1. AULA as a manufacturer

AULA is a Chinese peripheral brand whose keyboards are built on chips manufactured by **SinoWealth** (also referred to as SUOAI in some documentation). This is a common practice in the budget mechanical keyboard market — multiple brands (AULA, Royal Kludge, Redragon, etc.) buy from the same chip suppliers and ship very similar firmware stacks under different branding.

**Key implication:** reverse engineering work done on one AULA keyboard is very likely to carry over to other AULA models, including the L99.

---

## 2. USB identifiers — the VID/PID system

Every USB device has a **Vendor ID (VID)** and a **Product ID (PID)**. The VID identifies the manufacturer, the PID identifies the specific product.

From the OpenRGB issue tracker, the AULA F75 has been identified as:

```
VID: 258A
PID: 010C
```

**`VID 258A` appears to be shared across the entire AULA keyboard lineup.** This is the most important piece of information for the project — it means all AULA keyboards speak the same "language" at the USB level, even if some protocol details vary per model.

When the L99 arrives, the first step is to verify its VID/PID via Windows Device Manager. If VID is `258A`, the L99 is in the same family.

### How to check VID/PID on Windows
1. Open Device Manager
2. Find the keyboard under "Human Interface Devices"
3. Right-click → Properties → Details → Hardware IDs
4. Look for `VID_XXXX&PID_XXXX`

---

## 3. USB interface structure (typical AULA layout)

Based on research across multiple AULA models, the keyboard exposes **multiple USB interfaces simultaneously**:

| Interface | Purpose |
|---|---|
| Boot HID | Standard keypress events (works on any OS) |
| Media HID | Volume, play/pause, media shortcuts |
| Vendor Specific | Proprietary communication with the driver software |

The **Vendor Specific interface** is where everything interesting happens — RGB control, macro programming, and (on the L99) screen commands. This is the interface to target during reverse engineering.

---

## 4. Packet structure hypothesis

Based on analysis of similar AULA/SinoWealth keyboards:

- Commands are sent as **64-byte HID reports** (standard HID packet size)
- Data is sent as "blobs" — raw binary payloads encoding configuration or image data
- Some commands may include a **checksum** at the end for data integrity validation
- For image/screen data, the protocol likely uses **"tunneling"** — the same basic packet structure as RGB commands, but with much denser payloads to transfer pixel data

This is the same approach used by AIDA64 for pushing system stats to LCD panels on PC cases, and by the Stream Deck for its per-key displays.

---

## 5. Existing open-source projects

### AULA-specific

| Project | Model | What's done | Language | Link |
|---|---|---|---|---|
| aegis-2067usb | AULA F2067 | Full USB protocol reverse engineered, CLI tool | Rust | [GitHub](https://github.com/progzone122/aegis-2067usb-custom-software) |
| Aula-F87-Controller | AULA F87 | Protocol documented, Python CLI + WebHID app | Python / TypeScript | [GitHub](https://github.com/marcoslor/Aula-F87-Controller) |
| OpenRGB issue #4232 | AULA F75 | VID/PID identified, device capture requested | — | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/4232) |
| OpenRGB issue #5253 | AULA F108Pro | Device request opened | — | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/5253) |

### Royal Kludge (same VID family, similar protocol)

| Project | What's done | Link |
|---|---|---|
| Rangoli | Full USB protocol reverse engineered via Wireshark, cross-platform app | [GitHub](https://github.com/rnayabed/rangoli) |
| Kludge Knight | Browser-based WebHID configurator built on Rangoli's protocol research | [GitHub](https://github.com/vinc3m1/kludgeknight) |

**Key observation:** The official AULA driver and the official Royal Kludge driver are visually nearly identical. This strongly suggests a shared codebase or SDK from the chip manufacturer, meaning the protocol structures documented in Rangoli are very likely applicable — at least partially — to the L99.

---

## 6. The screen — what makes the L99 unique

Most AULA keyboards (F75, F87, F2067) have no screen. The L99 adds a **3.98" IPS touchscreen at 320×480 resolution**, which is the entire point of this project.

A few comparable keyboards with screens exist in the market:
- **Royal Kludge S98 / M87 / M75** — small OLED/TFT info displays (not touch, not interactive)
- **MiraBox K1W** (Kickstarter, 2026) — 6 individual LCD keys, browser-based Web UI, open SDK promised
- **MMOBIEL stream deck keyboard** — 6 LCD macro keys, uses Elgato Stream Deck software
- **Corsair K100 Air** — integrated Stream Deck keys, ~400€

**None of these combine a large touchscreen with an open software ecosystem.** The L99 has the hardware. This project aims to build the software.

---

## 7. Why the screen protocol is likely discoverable

The screen almost certainly communicates via the same **Vendor Specific USB interface** used for RGB and macros — just with larger payloads. The driver already demonstrates this works (it pushes GIFs and images to the screen). The question is just: what exact byte sequence triggers what behaviour?

Wireshark + USBPcap will capture every packet exchanged during these operations. Once captured, comparing them against the documented F87 protocol (marcoslor's `docs/PROTOCOL.md`) should reveal the pattern quickly.

---

## 8. People to potentially contact

| Who | Why | Where |
|---|---|---|
| marcoslor | Built the F87 controller, has deep protocol knowledge, same VID | [GitHub](https://github.com/marcoslor/Aula-F87-Controller) |
| rnayabed | Built Rangoli (RK reverse engineering), similar approach | [GitHub](https://github.com/rnayabed/rangoli) |
| progzone122 | Built aegis-2067usb for AULA F2067 | [GitHub](https://github.com/progzone122/aegis-2067usb-custom-software) |
| OpenRGB community | Already tracking multiple AULA devices | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB) |

---


*This document will be updated as the project progresses.*
