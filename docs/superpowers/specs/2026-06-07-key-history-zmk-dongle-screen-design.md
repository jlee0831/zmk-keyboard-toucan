# Key History Display — zmk-dongle-screen Port Design

**Date:** 2026-06-07
**Supersedes:** `2026-06-06-key-history-display-design.md`

## Background

The original key history display feature was built on top of `carrefinho/prospector-zmk-module`. The ZMK behavior drivers (toggle and scroll) never fired correctly, traced to incorrect use of `DT_INST_FOREACH_STATUS_OKAY_VARGS` (not available in ZMK v0.3). The prospector module also has broader known instability. This design ports the feature to `janpfischer/zmk-dongle-screen`, a more stable and actively maintained base, while adopting its native widget conventions.

## Goal

Add a scrollable key history screen to the toucan dongle display, accessible via `&kh_toggle`, showing physical key positions, resolved keycodes, active layers, and layer change events — implemented as a native widget in a fork of `zmk-dongle-screen`.

## Hardware

- Board: `seeeduino_xiao_ble`
- Display: ST7789V, 280×240 landscape, RGB565
- ZMK version: v0.3

## Architecture

### Thread model

All ring buffer reads and writes happen on the **display thread**, via `ZMK_DISPLAY_WIDGET_LISTENER` callbacks. This eliminates the cross-thread ring buffer mutex from the original implementation.

The only cross-thread state is `kh_is_recording` (atomic), written by behavior drivers on the ZMK main thread and read by listeners on the display thread.

Behavior drivers dispatch toggle and scroll actions to the display thread via `k_work_submit_to_queue(zmk_display_work_q(), ...)`, since `lv_scr_load()` must run on the display thread.

### Event listeners

Three `ZMK_DISPLAY_WIDGET_LISTENER` subscriptions, each running on the display thread:

| Subscription | Callback | Action |
|---|---|---|
| `zmk_position_state_changed` | `kh_on_position()` | Push `KH_KEY_PRESS` or `KH_KEY_RELEASE`; rebuild screen if active |
| `zmk_layer_state_changed` | `kh_on_layer()` | Push `KH_LAYER_CHANGE`; rebuild screen if active |
| `zmk_keycode_state_changed` | `kh_on_keycode()` | Call `kh_attach_keycode()`; rebuild screen if active |

All three check `kh_is_recording()` before touching the ring buffer.

### Behavior fix

The root cause of the original behavior-not-triggering bug was `DT_INST_FOREACH_STATUS_OKAY_VARGS`, which does not exist in ZMK v0.3. The corrected registration uses the per-instance macro pattern:

```c
#define KH_TOGGLE_INST(n) \
    DEVICE_DT_INST_DEFINE(n, NULL, NULL, NULL, NULL, \
        APPLICATION, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT, \
        &kh_toggle_driver_api);
DT_INST_FOREACH_STATUS_OKAY(KH_TOGGLE_INST)
```

Same pattern applies to the scroll behavior with a config struct.

## File Map

### New fork: `jlee0831/zmk-dongle-screen` (branch `feat/key-history`)

All shield files are under `boards/shields/dongle_screen/` unless noted.

| File | Action |
|---|---|
| `include/key_history.h` | Create — shared types and API |
| `src/widgets/key_history_widget.c` | Create — ring buffer + 3 ZMK_DISPLAY_WIDGET_LISTENER subscriptions + LVGL screen |
| `src/behaviors/key_history_behaviors.c` | Create — fixed behavior drivers |
| `dts/bindings/zmk,key-history-toggle.yaml` | Create — at fork root |
| `dts/bindings/zmk,key-history-scroll.yaml` | Create — at fork root |
| `Kconfig.shield` | Modify — add `DONGLE_SCREEN_KEY_HISTORY` symbol |
| `Kconfig.defconfig` | Modify — enable Montserrat 12 + 14 when feature on |
| `CMakeLists.txt` | Modify — compile new sources under `if(CONFIG_DONGLE_SCREEN_KEY_HISTORY)` |
| `src/custom_status_screen.c` | Modify — init key history screen at startup with `#if` guard |
| `zephyr/module.yml` | Modify — add `dts_root: .` |

### `toucan-zmk` repo (branch `feat/key-history-display`)

| File | Change |
|---|---|
| `config/west.yml` | Replace `prospector-zmk-module` remote + project with `zmk-dongle-screen` fork |
| `config/toucan.conf` | Replace `CONFIG_TOUCAN_KEY_HISTORY=y` → `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y`; remove prospector-specific options (`CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR`, `CONFIG_PROSPECTOR_FIXED_BRIGHTNESS`, `CONFIG_NICE_VIEW_WIDGET_INVERTED`) |
| `config/toucan.keymap` | Behavior DTS nodes unchanged; update shield name in build docs |

## Key Types (`key_history.h`)

```c
#define KH_RING_BUFFER_SIZE 64

typedef enum { KH_KEY_PRESS, KH_KEY_RELEASE, KH_LAYER_CHANGE } kh_event_type_t;

typedef struct {
    kh_event_type_t type;
    uint8_t  layer;
    uint32_t position;     /* 0xFFFFFFFF = unknown */
    uint32_t keycode;      /* 0 = unresolved */
    uint8_t  mods;
    uint8_t  new_layer;    /* KH_LAYER_CHANGE only */
    bool     layer_active; /* KH_LAYER_CHANGE only */
    uint32_t timestamp_ms;
} kh_entry_t;
```

Ring buffer API: `kh_push()`, `kh_count()`, `kh_get(newest_offset, out)`, `kh_attach_keycode()` — all display-thread-only, no mutex.

State API: `kh_set_recording(bool)` / `kh_is_recording()` — atomic. `kh_set_active(bool)` / `kh_is_active()` — atomic. `kh_set_scroll_offset()` / `kh_get_scroll_offset()` — display-thread-only.

Screen API: `kh_screen_create()`, `kh_screen_rebuild()`, `kh_set_screens(normal, history)`, `kh_get_screen()`, `kh_get_normal_screen()`.

Behavior API: `kh_cmd_toggle()`, `kh_cmd_scroll_up()`, `kh_cmd_scroll_down()` — submit work to display thread.

## LVGL Screen

280×240, same layout as original design:
- 32px dark header: "KEY HISTORY" title + "j/k=scroll  H=close" hint
- 208px list area: 10 visible rows × 20px each
- Columns: row index, key position, arrow, keycode, layer badge, age
- Layer change events rendered as green rows spanning keycode + badge columns
- Age-based opacity fading (100% < 2s, 60% < 4s, 30% < 6s, 15% older)
- Key releases not displayed

## Kconfig Symbol

`CONFIG_DONGLE_SCREEN_KEY_HISTORY` (bool, default n, depends on `SHIELD_DONGLE_SCREEN`).

When enabled: `LV_FONT_MONTSERRAT_12=y`, `LV_FONT_MONTSERRAT_14=y`.

## DTS Bindings

`zmk,key-history-toggle` — zero params, include `zero_param.yaml`.

`zmk,key-history-scroll` — zero params, include `zero_param.yaml`, one property: `direction` (int, required, 0=up/older, 1=down/newer).

## Build Command (unchanged)

```bash
west build -b seeeduino_xiao_ble -p -- \
  -DSHIELD="toucan_dongle rgbled_adapter dongle_screen" \
  -DZMK_CONFIG="/path/to/toucan-zmk/config" \
  -DSNIPPET="studio-rpc-usb-uart" \
  -DCONFIG_ZMK_STUDIO=y
```

Note: `prospector_adapter` → `dongle_screen` in the shield list.

## Verification

1. Build with `CONFIG_DONGLE_SCREEN_KEY_HISTORY=n` — existing widgets unchanged
2. Build with `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y` — compiles clean
3. Flash and confirm existing layer/battery widgets still work
4. Press `&kh_toggle` — history screen opens
5. Type keys — rows appear with correct positions and keycodes
6. Press `&kh_scroll_up` / `&kh_scroll_down` — offset scrolls
7. While history open, press keys — they do NOT appear after toggle off/on (recording paused)
8. Wait 5s, open history — older rows are visibly dimmer
