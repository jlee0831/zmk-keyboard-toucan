# Key History Display — Design Spec

**Date:** 2026-06-06
**Status:** Approved

## Goal

Show a scrollable history of recent key events on the Prospector dongle's display, accessible on demand via a keymap binding, so that unexpected keystrokes can be investigated after the fact — including which physical keys were pressed, what keycodes resulted, and which layers were active.

## Background

The Prospector dongle is always USB-connected and acts as the split keyboard's central unit. All key events from both halves flow through it before reaching the host. This makes it the ideal place to capture and display key history without requiring USB logging (which is blocked by a `studio-rpc-usb-uart` conflict documented in `scripts/keylogging-research.md`).

## Hardware

- **MCU:** Seeed Studio XIAO nRF52840
- **Display:** Waveshare 1.69" IPS LCD, ST7789 driver, 240×280 pixels physical, **mounted in landscape orientation** (LVGL coordinate space: 280×240)
- **Color depth:** 16-bit RGB565
- **UI framework:** LVGL with Montserrat 20 as default font
- **Touch:** CST816S panel present on hardware but no Zephyr driver implemented — not used

## Architecture

All new code is added to a fork of `carrefinho/prospector-zmk-module`. The `toucan-zmk` config repo requires three small changes only.

### Changes to `prospector-zmk-module` fork

New files added under `boards/shields/prospector_adapter/`:

| File | Purpose |
|---|---|
| `src/key_history_listener.c` | ZMK event listener; maintains ring buffer |
| `src/widgets/key_history_screen.c` | LVGL history screen; renders ring buffer contents |
| `src/behaviors/key_history_behaviors.c` | Three custom ZMK behaviors: toggle, scroll up, scroll down |
| `include/key_history.h` | Shared types and API between the above |
| `dts/bindings/zmk,key-history-toggle.yaml` | DTS binding for `&kh_toggle` |
| `dts/bindings/zmk,key-history-scroll.yaml` | DTS binding for `&kh_scroll_up` / `&kh_scroll_down` |

`src/custom_status_screen.c` gets one addition: it initializes the history screen at startup alongside the existing normal screen.

`CMakeLists.txt` is updated to compile the new source files when `CONFIG_TOUCAN_KEY_HISTORY=y`.

`Kconfig.defconfig` adds `config TOUCAN_KEY_HISTORY bool`, defaulting to `n`.

### Changes to `toucan-zmk` config repo

1. `config/west.yml` — update `prospector-zmk-module` remote to point at the fork
2. `config/toucan.conf` (dongle-specific conf, or new `toucan_dongle.conf`) — add `CONFIG_TOUCAN_KEY_HISTORY=y`
3. `config/toucan.keymap` — add three behavior instances and bind them

## Event Capture

### Subscribed events

| ZMK event | Data used | Why |
|---|---|---|
| `zmk_position_state_changed` | `position`, `state` (pressed/released), timestamp | Physical key that was pressed |
| `zmk_layer_state_changed` | `layer`, `state` (active/inactive) | Layer activation/deactivation |
| `zmk_keycode_state_changed` | `keycode`, `implicit_modifiers`, `explicit_modifiers`, `state` | Actual HID output sent to the host |

When a `zmk_keycode_state_changed` event fires, the listener attaches its keycode to the most recent `KH_KEY_PRESS` entry in the ring buffer that has `keycode == 0` (not yet resolved). In ZMK, the position event and its resulting keycode event are emitted in the same synchronous processing cycle, so this heuristic is reliable in practice. If no unresolved press exists (e.g. the keycode came from a macro or repeat behavior with no single position trigger), the keycode is stored as a standalone `KH_KEY_PRESS` entry with `position == 0xFFFF` to indicate no physical position.

### Ring buffer

- **Size:** 64 entries (power of 2; fits in ~2 KB RAM)
- **Entry fields:**

```c
typedef enum { KH_KEY_PRESS, KH_KEY_RELEASE, KH_LAYER_CHANGE } kh_event_type_t;

typedef struct {
    kh_event_type_t type;
    uint8_t         layer;       // active layer at time of event
    uint16_t        position;    // key position (KH_KEY_PRESS / KH_KEY_RELEASE)
    uint16_t        keycode;     // resolved HID keycode (0 if none)
    uint8_t         mods;        // active modifiers at time of press
    uint8_t         new_layer;   // target layer (KH_LAYER_CHANGE only)
    uint32_t        timestamp_ms;
} kh_entry_t;
```

- **Thread safety:** ring buffer is written from ZMK's main event thread and read from the LVGL display thread. A Zephyr `k_mutex` protects concurrent access.

### Recording pause

While the history screen is open (`history_active == true`), the event listener skips all recording. This prevents history-navigation key presses (toggle, scroll) from appearing in the buffer and keeps the buffer frozen at the state it was in when the screen was opened.

Recording resumes the moment the screen closes.

## Display

### Two LVGL screens

The Prospector has two LVGL screens that coexist in memory:

- **Normal screen** — the existing Prospector status screen (layer roller, battery bar, connection status widgets). Completely unchanged. Continues to receive ZMK widget updates in the background regardless of which screen is visible.
- **History screen** — created lazily on first open and reused. Displays the ring buffer contents.

Switching is done with `lv_scr_load()` from the LVGL work queue thread (never directly from a behavior or ISR).

### History screen layout (landscape 280×240)

```
┌─────────────────────────────────────────────┐  ← 280px wide
│  KEY HISTORY              ↑↓ scroll · H=close │  32px header
├─────────────────────────────────────────────┤
│ 1  #29  →  A            [BASE]        0.2s  │
│ 2  #30  →  SPACE        [BASE]        0.5s  │
│ 3  ◆    →  layer → SYM  [SYM ]        0.6s  │  ← green row
│ 4  #15  →  ENTER        [SYM ]        0.9s  │
│ 5  #16  →  BSPC         [SYM ]        1.2s  │
│ 6  ◆    →  layer → BASE [BASE]        1.5s  │
│ 7  #44  →  ESC          [BASE]        2.0s  │  ← fading
│ 8  #31  →  B            [BASE]        2.4s  │  ← fading
│ 9  #11  →  TAB          [BASE]        3.0s  │  ← very faint
│ 10 #05  →  CMD+C        [BASE]        3.4s  │  ← very faint
├─────────────────────────────────────────────┤
```
208px list area, ~10 rows visible at once (20px per row). Scroll reveals up to 64 entries.

### Visual treatment

- **Background:** black (`#000000`)
- **Header:** dark (`#0d1117`), white title, dim scroll hint
- **Key press rows:** white keycode text, purple position (`#7c6af7`), layer badge (color varies by layer index), dim timestamp
- **Layer change rows:** green highlight (`#0d1a0d` background, `#3fb950` text)
- **Age fading:** age is computed relative to the most recent entry's timestamp (not wall clock time), so opening history long after the last keypress doesn't show everything fully faded. Entries more than ~2s older than the most recent begin to fade; entries more than ~5s older are very dim.
- **Key releases:** stored in ring buffer but not displayed by default (excluded from the rendered list to reduce noise)

### Layer badge colors

| Layer | Color |
|---|---|
| BASE (0) | Gray `#21262d` |
| SYM (1) | Blue `#1f2d3d` / `#58a6ff` |
| NUM (2) | Purple `#2d1f3d` / `#bc8cff` |
| FNC (3) | Orange `#2d1d0d` / `#f0883e` |

## Interaction Model

### Toggle model

```
press hyper+H  →  history screen opens, list scrolled to most-recent entry
press hyper+K  →  scroll up (toward older entries)
press hyper+J  →  scroll down (toward newer entries)
press hyper+H  →  history screen closes, normal status screen resumes
```

### Behaviors

Three custom ZMK behaviors are registered as DTS nodes in `toucan.keymap`:

```dts
/ {
    behaviors {
        kh_toggle: kh_toggle {
            compatible = "zmk,key-history-toggle";
            #binding-cells = <0>;
        };
        kh_scroll_up: kh_scroll_up {
            compatible = "zmk,key-history-scroll";
            #binding-cells = <0>;
            direction = <KH_SCROLL_UP>;
        };
        kh_scroll_down: kh_scroll_down {
            compatible = "zmk,key-history-scroll";
            #binding-cells = <0>;
            direction = <KH_SCROLL_DOWN>;
        };
    };
};
```

Suggested keymap bindings (user adjusts to taste):

| Key | Behavior |
|---|---|
| `hyper+H` | `&kh_toggle` |
| `hyper+K` | `&kh_scroll_up` |
| `hyper+J` | `&kh_scroll_down` |

### Transparent scroll behaviors

`&kh_scroll_up` and `&kh_scroll_down` are **transparent when history is closed**: they pass through as normal keycodes to the host. This means `hyper+K` and `hyper+J` can remain useful bindings in normal usage; they only change meaning while history mode is active.

### Screen switch threading

ZMK behaviors execute on the main thread; LVGL runs on its dedicated display thread. The toggle behavior does not call LVGL directly. Instead it submits a `k_work` item to the LVGL work queue, which then calls `lv_scr_load()`. This is the same pattern used by existing ZMK display widgets.

## Out of Scope

- Touch-based scrolling (CST816S not implemented in ZMK)
- Host-side keylogger (addressed by this feature)
- Filtering history by key position or layer (future enhancement)
- Exporting history over USB (future enhancement)
