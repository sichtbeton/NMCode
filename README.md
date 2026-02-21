# NMCode

Firmware for **NOISE Machine (NMSVE)** built on **ESP32 + BLE-MIDI**.

This project turns the hardware controls (12 buttons, slider, and rotary) into a wireless MIDI controller that can pair with iPad/iPhone/macOS apps (for example Koala Sampler), and other BLE-MIDI hosts.

---

## What this firmware does

- Advertises as a BLE-MIDI peripheral (`NOISE-XXXX`).
- Sends MIDI **Note On/Off** from 12 buttons.
- Uses a **4-zone slider mapping** so each button can trigger different note pages.
- Sends rotary **MIDI CC** values continuously while turning.
- Supports a **modifier CC mode**: when any button is held, rotary sends a button+zone specific CC number.
- Supports **boot-held MIDI channel select** with buttons **1..12**, and saves selection in NVS (`Preferences`).
- Uses onboard LEDs for connection and mode feedback.

---

## Hardware mapping

### GPIO

- Blue LED: `GPIO14`
- Green LED: `GPIO4`
- Slider ADC: `GPIO36` (`ADC1_CH0`)
- Rotary ADC: `GPIO39` (`ADC1_CH3`)
- Buttons 1..12: `16,17,18,21,19,25,22,23,27,26,35,34`

### Slider zones

Slider is converted to percent (0..100) and mapped:

- Zone 0: `< 14%`
- Zone 1: `14% .. <55%`
- Zone 2: `55% .. <75%`
- Zone 3: `>= 75%`

Green LED reflects zone parity (`on` in zones 0 and 2).

---

## MIDI behavior

### Channel

- Runtime status bytes are built from selected MIDI channel:
  - Note On: `0x90 | (ch-1)`
  - Note Off: `0x80 | (ch-1)`
  - CC: `0xB0 | (ch-1)`
- Supported channels: **1..12**.

### Button notes

- `NOTE_BASE = 36`
- `NOTES_PER_ZONE = 12`
- Note number per button:

```text
note = 36 + (zone * 12) + buttonIndex
```

So each slider zone gives a 12-note page.

### Rotary CC

Default rotary CC when no button is held:

- `CC = 100`

Modifier CC when a button is held:

```text
CC = 10 + (zone * 12) + buttonIndex
```

This allows zone-dependent per-button rotary modulation targets.

---

## BLE-MIDI details

- Service UUID: `03b80e5a-ede8-4b33-a751-6ce34ec4c700`
- Characteristic UUID: `7772e5db-3868-4112-a1a9-f2669d106bf3`
- Characteristic properties: `NOTIFY | READ | WRITE_NR`
- MTU set to `185`
- TX power: `ESP_PWR_LVL_P7`
- Advertising interval: min `160`, max `240`

MIDI events are framed in BLE-MIDI timestamped packets (5 bytes per MIDI event in current builder flow).

---

## Boot-held MIDI channel selection (1..12)

At startup, firmware checks button state for ~600 ms:

- Hold button `N` during boot to select MIDI channel `N`.
- Selection is persisted in `Preferences` namespace `noise`, key `midich`.
- Green LED turns solid briefly (~500 ms) when a held channel is stored.
- If no boot-held button is detected, saved channel is used.

---

## LED behavior

- **Blue LED**
  - Blinks while not connected over BLE.
  - Solid on when connected.
- **Green LED**
  - Zone indicator during runtime (zones 0/2 on, 1/3 off).
  - Startup confirmation pulse when saving boot-held channel.

---

## Build / flash

## Prerequisites

- Arduino IDE with ESP32 core:
  - https://github.com/espressif/arduino-esp32
- USB-to-TTL adapter for programming the board.

### Board setup

- Select board profile: **FireBeetle-ESP32**.
- To enter programming mode, pull `BOOT` to `GND` on startup.

Programming header pin order (top to bottom):

`BOOT, EN, GND, 3V3, RX, TX`

Reference diagram:

![Programming pins](https://github.com/thisisnoiseinc/NMCode/blob/main/Programming/Pins.png)

---

## Pairing & basic test (Koala / iOS)

1. Power device and wait for BLE advertisement (`NOISE-XXXX`).
2. In host app (Koala standalone, AUM, etc.), connect to the BLE MIDI device.
3. Verify Blue LED becomes solid.
4. Press buttons and confirm notes trigger.
5. Move slider, then press same button to confirm zone note page changes.
6. Turn rotary:
   - no button held: CC 100
   - button held: per-button/zone modifier CC

If host sees the device but no MIDI arrives, see FAQ below.

---

## Technical notes (for maintainers)

- Debounce uses signed integrator counters (`DEB_ON/DEB_OFF`) scanned every `BTN_SCAN_MS`.
- Release guard (`RELEASE_GUARD`) protects against rapid false-off transitions.
- Rotary path applies nonlinear response (`ROT_CURVE`) and movement gating (`FAST_THRESH`, `MED_THRESH`, `ROT_DEADBAND`, `CC_GAP_MS`).
- BLE disconnect callback restarts advertising automatically.
- Device name includes lower 16 bits of eFuse MAC (`NOISE-%04X`).

---

## Minor FAQ / possible improvements

### 1) “Device connects, but no sound in app”

- Confirm app MIDI input routing (some apps require explicit input assignment per track/pad).
- Confirm channel expected by app matches selected channel.
- Reboot while holding button 1 to force known channel 1.

### 2) “Rotary feels too filtered / not snappy enough”

Potential tuning constants:

- `ROT_CURVE`
- `CC_GAP_MS`
- `FAST_THRESH`
- `MED_THRESH`
- `ROT_DEADBAND`
- `ROT_EMA_A`

Lower filtering values increase responsiveness, but can increase jitter.

### 3) “Need wider MIDI channel support (1..16)”

Current implementation intentionally clamps to 1..12. Extending to 16 is straightforward in `setMidiChannel()` and persistence validation.

### 4) “Can I expose configurable mappings?”

Yes. A practical next step is a compile-time profile table (or NVS-backed map) for:

- note base
- zone cuts
- default rotary CC
- modifier CC formula

### 5) “Can this be made more DAW-friendly?”

Possible improvements:

- fixed channel/preset profiles for common DAWs
- optional high-resolution CC strategy
- long-press combos for quick setup modes

---

## Credits

Originally based on work from:

- https://github.com/neilbags/arduino-esp32-BLE-MIDI
