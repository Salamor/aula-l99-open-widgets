# Research & Prior Art

This document collects all research gathered before and during the reverse engineering process. It serves as a reference for contributors and a record of why certain technical decisions were made.

---

## 1. AULA as a manufacturer

AULA is a Chinese peripheral brand. Most of their keyboards use chips from **SinoWealth** (`VID 258A`) — a common practice in the budget mechanical keyboard market (AULA, Royal Kludge, Redragon, etc. all share similar chipsets and firmware stacks).

**Critical correction for the L99:** The L99 does NOT use SinoWealth. It uses a **SONiX** chip (`VID 0C45`), the same as the AULA F75 MAX. This is confirmed by Device Manager on the day of hardware reception. The VID `258A` assumption documented elsewhere in the AULA community does not apply to this model.

---

## 2. USB identifiers — confirmed for the L99

### The L99 exposes TWO separate USB devices

When connected via USB-C, the L99 registers as two distinct USB devices simultaneously:

| Device | VID | PID | Type | Controls |
|---|---|---|---|---|
| Keyboard | `0C45` (SONiX) | `800A` | USB HID | Keypresses, RGB, macros, profiles |
| Screen | `EEEF` | `268A` | USB Serial (COM port) | Display content |

This is the most important architectural finding of the project. The screen is **not** controlled through the keyboard's HID interface — it has its own dedicated USB serial connection that appears in Windows as a virtual COM port (typically COM3).

### The F75 MAX connection

The keyboard side (`0C45:800A`) is **identical** to the AULA F75 MAX. This means Simon-Martens' work in `F75_Initializer` is directly applicable to the L99's keyboard commands (RGB, RTC sync, etc.), even though the screen architectures differ between the two models.

### Other AULA models (SinoWealth family, VID 258A)

The majority of AULA keyboards use `VID 258A`:
- F75 (non-MAX): `258A:010C`
- F87, F87 Pro, F2067, F99, etc.

The Royal Kludge RK61 also uses `VID 258A` (`004a`/`007a`), confirming the AULA/RK shared chipset — but this applies to SinoWealth models only, not the L99.

---

## 3. The screen USB device — EEEF:268A

### What it is

The screen registers as a **USB Serial Device (COM port)** in Windows Device Manager, manufactured by Microsoft (standard CDC serial driver). This is very different from HID: instead of structured feature reports, the host simply writes a stream of bytes into the port.

This is the same architecture used by "Turing Smart Screen" / USB monitoring displays — but with a completely different protocol.

### VID EEEF — the OEM origin

`VID EEEF` is the factory/OEM identifier used by the chip manufacturer that also supplies:
- The **Ajazz AKP153** and **Mirabox HSV293S** (stream deck clones)
- The **Turing Smart Screen** family of USB monitoring displays

However: the protocol is NOT shared. AULA uses `PID 268A` and a completely custom serial protocol, while Turing Smart Screen uses `ef 69` magic bytes. The VID is the same OEM chip supplier, but AULA wrote their own firmware.

### Search result: no prior art

An exhaustive search for the magic bytes `12 34 56 78 02` across GitHub, forums, and protocol databases found **zero matches**. This protocol has never been publicly documented before. Our Wireshark captures are the first public record.

---

## 4. Screen protocol — findings from Wireshark captures

### Captures available

Two captures were made on the day of hardware reception, using Wireshark + USBPcap on a Windows 10 VM with USB passthrough:

| File | Image sent | Resolution |
|---|---|---|
| `captures/capture_red_picture.pcapng` | Solid red rectangle | 320 × 480 px |
| `captures/capture_complex_picture.pcapng` | Complex image | 320 × 480 px |

### Image transfer protocol (confirmed)

The image is sent as a **single bulk transfer** over the COM serial port. The transfer is structured as:

```
[64-byte header] + [JPEG data]
```

**Header structure (64 bytes):**

| Bytes | Value (red capture) | Meaning |
|---|---|---|
| 00–03 | `12 34 56 78` | Magic signature |
| 04 | `02` | Command type: send image |
| 05–07 | `00 00 00` | Padding |
| 08–11 | `E0 01 00 00` | Height = 480px (little-endian uint32) |
| 12–15 | `E0 01 00 00` | Width = 480px (little-endian uint32) |
| 16–55 | `00 × 40` | Padding (40 zero bytes) |
| 56–59 | `02 00 00 00` | Unknown — possibly type/quality indicator |
| 60–63 | `34 39 01 00` | JPEG size = 79156 bytes (little-endian uint32) |

**JPEG data:** Standard JFIF JPEG, unmodified, begins immediately at byte 64 with `FF D8 FF E0`.

### Refresh trigger (partially identified)

After the complete image transfer, a **HID SET_REPORT** packet is sent to the keyboard device (`0C45:800A`) — not to the screen device. This is the signal that causes the screen to display the newly received image. The exact bytes of this packet are identified in the capture but need further verification.

### What the protocol tells us in Python pseudocode

```python
import struct, serial, hid

# 1. Prepare header
width, height = 320, 480
jpeg_data = open("widget.jpg", "rb").read()

magic    = b'\x12\x34\x56\x78\x02'
padding1 = b'\x00' * 3
dims     = struct.pack('<II', height, width)
padding2 = b'\x00' * 40
unknown  = struct.pack('<I', 2)
size     = struct.pack('<I', len(jpeg_data))

header = magic + padding1 + dims + padding2 + unknown + size
# header is exactly 64 bytes

# 2. Send to screen via COM port
with serial.Serial('COM3') as ser:
    ser.write(header + jpeg_data)

# 3. Trigger refresh via HID (exact bytes TBD from capture)
# keyboard = hid.device()
# keyboard.open(0x0C45, 0x800A)
# keyboard.send_feature_report([...])
```

### What still needs to be captured

| Interaction | Why it matters |
|---|---|
| **Tapping the screen** | Touch events — coordinates X/Y, format, which device sends them |
| **Changing screen brightness** | Reveals brightness command format |
| **Screen on/off** | Reveals power control commands |
| **Navigating the official driver UI** | May reveal additional command types |
| **RGB change on the keyboard** | Maps the HID command space for `0C45:800A` |

---

## 5. Keyboard protocol — the 0C45:800A side

### Connection to F75 MAX

The keyboard exposes multiple HID interfaces simultaneously (standard for this chip family):

| Interface | Purpose |
|---|---|
| Boot HID | Standard keypress events |
| Media HID | Volume, play/pause, media shortcuts |
| Vendor Specific (`UsagePage 0xFF00`) | RGB, macros, profiles, screen refresh trigger |

Simon-Martens documented the F75 MAX (same VID/PID) keyboard protocol:
- Commands: 64-byte HID feature reports
- Checksum: `sum(bytes[0:31]) & 0xFF`
- 4-packet initialization sequence for screen
- RTC sync command structure

This work is directly applicable to the L99 keyboard side. **Simon-Martens is the highest priority contact** for this project.

---

## 6. Existing open-source projects

### AULA-specific — Rank S (most directly applicable)

| Project | Model | Why relevant | Language | Link |
|---|---|---|---|---|
| F75_Initializer | AULA F75 MAX | **Same VID/PID as L99** (`0C45:800A`), controls screen from Linux, has pcapng captures | Python | [GitHub](https://github.com/Simon-Martens/F75_Initializer) |
| aula-rgb-controller | AULA F87 TK | Full RGB controller, 191 commits, protocol in `tools/protocol_notes.md`, GTK4 GUI | C | [GitHub](https://github.com/veysiemrah/aula-rgb-controller) |
| win-68-he-tool | AULA WIN60/68 HE | Copy of official AULA WebHID driver JS — readable source code of the protocol | JavaScript | [GitHub](https://github.com/caioalonso/win-68-he-tool) |
| OpenRGB issue #5166 | AULA F99 | Full wireless protocol RE'd by rodrigost23, merged into OpenRGB | — | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5166) |
| NollieL/SignalRgb_CN_Key | AULA series | Full AULA protocol in readable JS, `Aula Series.js` | JavaScript | [GitHub](https://github.com/NollieL/SignalRgb_CN_Key) |

### AULA-specific — Rank A (solid references)

| Project | Model | What's done | Language | Link |
|---|---|---|---|---|
| Aula-F87-Controller | AULA F87 | Protocol documented, Python CLI + WebHID, captures in `/captures` | Python/TS | [GitHub](https://github.com/marcoslor/Aula-F87-Controller) |
| aula-f87pro | AULA F87 Pro | CLI Python, RGB on Linux, VID `258A:010C` confirmed | Python | [GitHub](https://github.com/Ahorts/aula-f87pro) |
| aula_contol-f87 | AULA F87 | Jupyter notebook analysis, active RE | Python/C | [GitHub](https://github.com/umesh70/aula_contol-f87) |
| aegis-2067usb | AULA F2067 | Full protocol, CLI tool | Rust | [GitHub](https://github.com/progzone122/aegis-2067usb-custom-software) |
| tarantula | AULA (generic) | Early-stage Rust library | Rust | [GitHub](https://github.com/sangwon090/tarantula) |
| aula-f75-configurator | AULA F75 | WebHID configurator in progress | TypeScript | [GitHub](https://github.com/AmoabaKelvin/aula-f75-configurator) |

### AULA — OpenRGB tracker

| Issue | Model | Status | Link |
|---|---|---|---|
| #5166 | AULA F99 | Fully documented, merged | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5166) |
| #4232 | AULA F75 | VID/PID identified | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/4232) |
| #5253 | AULA F108Pro | Device request opened | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/5253) |

### USB screen ecosystems (reference architecture)

| Project | Protocol | Magic bytes | Relevance |
|---|---|---|---|
| [turing-smart-screen-python](https://github.com/mathoudebine/turing-smart-screen-python) | Serial COM | `ef 69` | Same architecture, different protocol |
| [mirajazz](https://github.com/4ndv/mirajazz) | USB HID | N/A | Ajazz AKP153/AKP815 — architectural reference |
| [ajazz-sdk](https://github.com/mishamyrt/ajazz-sdk) | USB HID | N/A | Rust SDK for Ajazz stream docks |

### Royal Kludge (SinoWealth family — not applicable to L99 hardware but useful for general AULA context)

| Project | What's done | Link |
|---|---|---|
| Rangoli | Full RK protocol via Wireshark, cross-platform app | [GitHub](https://github.com/rnayabed/rangoli) |
| Kludge Knight | Browser WebHID configurator | [GitHub](https://github.com/vinc3m1/kludgeknight) |
| rkcu | RK61 Python CLI, VID `258A:004a` confirmed | [GitHub](https://github.com/oddlyspaced/rkcu) |

---

## 7. The L99 in the keyboard-with-screen landscape

The market has several keyboards with integrated screens. Understanding how they differ helps clarify why the L99 is uniquely interesting and where our work sits.

| Keyboard | Screen size | Screen type | PC controls it | Protocol | RE status |
|---|---|---|---|---|---|
| **AULA L99** | 3.98" | Full canvas, touch | Yes — real-time JPEG via COM | Serial `EEEF:268A`, magic `12 34 56 78` | 🔶 This project |
| Ajazz AKP815 | 4.95" | Grid of 15 LCD buttons, touch | Yes — per-button 126×126px HID | USB HID `0300:XXXX` | ✅ `ajazz-sdk` |
| Ajazz AKP846 | 10.1" | Full extended display | Yes — OS-level monitor | DisplayLink or equivalent | ✅ Plug and play |
| Ajazz AK650 | 0.85" | Info display, no touch | Indirect (GIF upload only) | Proprietary, stored in keyboard | ❌ Closed |
| Ajazz AK35I V3 MAX | 1.14" | Info display, no touch | Indirect (GIF upload only) | Proprietary, stored in keyboard | ❌ Closed |
| AULA F75 MAX | Small | Info display, no touch | Yes — via keyboard HID interface | `0C45:800A` HID | 🔶 F75_Initializer |

**Key distinction:** The AKP815 treats the screen as a grid of addressable buttons (each gets its own image, touch returns a button number). The L99 receives a single full-resolution JPEG for the entire screen and presumably returns raw X/Y coordinates for touch events. This gives the L99 complete layout freedom — the software defines the UI, not the hardware.

---

## 8. AULA WebHID drivers — an inspectable entry point

AULA has published browser-based WebHID drivers for their Hall Effect keyboard lines. These are fully readable in browser DevTools:

| Model | URL |
|---|---|
| WIN60/68HE Standard | [device.aulacn.com](https://device.aulacn.com) |
| WIN60/68HE PRO/MAX | [win.aulacn.com](https://win.aulacn.com) |
| HERO68 Standard | [hero.aulastar.com](https://hero.aulastar.com) |
| HERO68 ULTRA / PRO/MAX | [magnet.aulastar.com](https://magnet.aulastar.com) |

`caioalonso` archived a local copy of the WIN60/68HE driver: [github.com/caioalonso/win-68-he-tool](https://github.com/caioalonso/win-68-he-tool). This is the most accessible entry point for understanding how AULA structures HID communication without hardware.

---

## 9. Community discussions

### r/aula — dorok15
- Standard USB HID feature reports confirmed (not locked-down)
- SinoWealth chipset confirmed for most AULA models
- Firmware: **SH68F073 (8051 MCU)** — *Note: this applies to SinoWealth models, L99 uses SONiX*
- macOS/Linux replacement app considered feasible

### r/AskReverseEngineering — umesh70
Author of [aula_contol-f87](https://github.com/umesh70/aula_contol-f87):
- Identified Interface 1 with UsagePage 0xff00 as RGB control channel
- Has HID descriptor dump
- Actively looking for collaborators

---

## 10. People to contact (priority order)

| Priority | Who | Why | Where |
|---|---|---|---|
| 🥇 1 | **Simon-Martens** | **Same keyboard hardware** (`0C45:800A`), already has screen init code for F75 MAX from Linux. Fastest path to keyboard-side commands. | [GitHub](https://github.com/Simon-Martens/F75_Initializer) |
| 🥈 2 | **veysiemrah** | Most complete AULA open-source project (191 commits), full protocol docs | [GitHub](https://github.com/veysiemrah/aula-rgb-controller) |
| 🥈 2 | **rodrigost23** | RE'd the complete F99 wireless protocol, submitted working OpenRGB PRs | [GitLab](https://gitlab.com/rodrigost23) |
| 🥉 3 | **marcoslor** | Built F87 controller, has captures, already starred this repo | [GitHub](https://github.com/marcoslor/Aula-F87-Controller) |
| 🥉 3 | **umesh70** | Active F87 RE, looking for collaborators, same frustration | [GitHub](https://github.com/umesh70/aula_contol-f87) |
| 🥉 3 | **darkdex52** | F75 MAX user with Wireshark captures, opened OpenRGB issue #5326 | [GitLab](https://gitlab.com/darkdex52) |
| 4 | **rnayabed** | Built Rangoli (RK RE), similar approach | [GitHub](https://github.com/rnayabed/rangoli) |
| 4 | **progzone122** | Built aegis-2067usb for AULA F2067 | [GitHub](https://github.com/progzone122/aegis-2067usb-custom-software) |
| 4 | **OpenRGB community** | Already tracking multiple AULA devices | [GitLab](https://gitlab.com/CalcProgrammer1/OpenRGB) |

---

*This document will be updated as captures are added and the protocol is mapped further.*
