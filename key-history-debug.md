# Key History Display — Debug State

Last updated: 2026-06-07

## What This Feature Is

A scrollable key history screen on the Prospector dongle display (280×240 ST7789, landscape). Toggle in/out via keymap. Shows recent keypresses with physical position, resolved keycode, active layer badge, and age-faded rows.

## Repo State

Both branches are pushed to GitHub and CI has built from them.

| Repo | Branch | Latest commit |
|------|--------|---------------|
| `jlee0831/prospector-zmk-module` | `feat/key-history-display` | `ec840c4` + logging |
| `jlee0831/toucan-zmk` | `feat/key-history-display` | `365ea0d` + dongle.conf + keymap |

### toucan-zmk commits (in order)
- `61eed65` — Add key history display design spec
- `365ea0d` — feat: wire up key history display feature
- *(uncommitted)* — add `config/toucan_dongle.conf`, add `&kh_toggle` to NUM layer pos 40

### prospector-zmk-module commits (in order)
- `551ecb1` — initial implementation
- `d76fd52` — fix: `LV_OPA_15` undefined, `KH_VISIBLE_ROWS` out of scope
- `92fa34b` — fix: increase LVGL pool and display stack (broken — see below)
- `ec840c4` — fix: correct Kconfig default ordering (pushed and built, still not working)
- *(uncommitted)* — add diagnostic logging to `key_history_behaviors.c` and `custom_status_screen.c`

## Symptom

Pressing the FNC layer left thumb cluster does nothing. No screen change visible on the dongle.

## Root Cause Analysis (2026-06-07)

### Most Likely: CONFIG_TOUCAN_KEY_HISTORY not applied to dongle build

ZMK's `build-user-config.yml@v0.3` applies user conf files by matching **shield name**. The dongle build uses shields `toucan_dongle rgbled_adapter prospector_adapter`. The user's `config/` directory has `toucan.conf` (with `CONFIG_TOUCAN_KEY_HISTORY=y`) but NO `toucan_dongle.conf`, `rgbled_adapter.conf`, or `prospector_adapter.conf`.

This means `toucan.conf` may never be applied to the dongle build. If `CONFIG_TOUCAN_KEY_HISTORY=n` (the default in `Kconfig.shield`):
- `key_history_behaviors.c` is not compiled (behind CMake `if (CONFIG_TOUCAN_KEY_HISTORY)`)
- Behavior device is never registered
- `zmk_behavior_get_binding("kh_toggle")` returns NULL at runtime → silently does nothing
- The build still succeeds (ZMK resolves behavior names as strings, not compile-time pointers)

This perfectly matches the symptom: normal screen works, toggle key does nothing, no crash.

**Fix applied**: Created `config/toucan_dongle.conf` with `CONFIG_TOUCAN_KEY_HISTORY=y`.

### Second-order issue: If the conf was applied but pool was too small

Previously confirmed that the `default 24000 if TOUCAN_KEY_HISTORY` Kconfig ordering was broken and fixed in `ec840c4`. After that fix, if `CONFIG_TOUCAN_KEY_HISTORY=y` was actually reaching the dongle build, the pool should be 24KB.

The logging added will confirm: if `KH: feature enabled, hist=<non-null>` appears on boot, then both the config and pool are working.

## Changes Made (2026-06-07)

### `jlee0831/toucan-zmk`

**`config/toucan_dongle.conf`** (new):
```
CONFIG_TOUCAN_KEY_HISTORY=y
```
Ensures `CONFIG_TOUCAN_KEY_HISTORY=y` is applied to the dongle build regardless of how ZMK resolves `toucan.conf`.

**`config/toucan.keymap`**: Added `&kh_toggle` to NUM layer thumb row (position 40, right inner thumb):
```
&trans  &mkp LCLK  &mkp RCLK  &trans  &kh_toggle  &kp NUMBER_0
```
Test sequence: hold left `&mo_tog 2 2` (position 36) to enter NUM, then press right inner thumb (position 40).
This isolates whether the FNC sticky layer sequence was the problem.

### `jlee0831/prospector-zmk-module`

**`src/behaviors/key_history_behaviors.c`**: Added `LOG_MODULE_REGISTER(kh_behaviors, LOG_LEVEL_WRN)`, plus:
- `LOG_WRN("kh_toggle: pressed")` in `kh_toggle_pressed()`
- `LOG_WRN("do_toggle: now_active=%d hist=%p", now_active, kh_get_screen())` in `do_toggle()`

**`src/custom_status_screen.c`**: Added inside the `#if IS_ENABLED(CONFIG_TOUCAN_KEY_HISTORY)` block:
- `LOG_WRN("KH: feature enabled, hist=%p", (void *)hist)`

## How to Interpret Results After Next Build/Flash

### Scenario A: Screen switches when pressing NUM+pos40 → success (root cause was the conf file)
The FNC binding may still work. If it does, root cause was the missing `toucan_dongle.conf`. If only NUM works but FNC still doesn't, the FNC sticky sequence has a problem (see hypothesis #3 below).

### Scenario B: RTT log shows "KH: feature enabled, hist=0x0" → LVGL pool still too small
`kh_screen_create()` is returning NULL. The pool increase is not taking effect. Check the `.config` artifact for `CONFIG_LV_Z_MEM_POOL_SIZE`.

### Scenario C: RTT log shows "kh_toggle: pressed" but NOT "do_toggle: ..." → work queue issue
`k_work_submit_to_queue(zmk_display_work_q(), ...)` is failing or the display queue is not processing. Investigate `zmk_display_work_q()` availability.

### Scenario D: "do_toggle: now_active=1" appears but screen doesn't change → display override
Something is calling `lv_scr_load(status_screen)` after `do_toggle()`. ZMK Studio is a candidate (the dongle uses `CONFIG_ZMK_STUDIO=y`).

### Scenario E: No RTT logs at all → logging not enabled or RTT not connected
The feature may still work (test by observing the display). If the screen DOES switch, logs just aren't visible.

## What Has Been Ruled Out

- **Compile errors**: All known errors (`LV_OPA_15`, `KH_VISIBLE_ROWS` scope) are fixed. Build succeeds.
- **Behavior code logic**: `do_toggle()`, `kh_toggle_pressed()`, screen create/rebuild all look correct on inspection.
- **ZMK screen override**: ZMK's display subsystem does NOT call `lv_scr_load()` on widget updates — only once at startup. `do_toggle()`'s `lv_scr_load()` should persist.

## FNC Layer Activation Sequence (for reference)

FNC (layer 3) is only reachable via `&sl 3` in the NUM layer. `&sl` is a one-shot sticky layer.

```
1. Hold left mo_tog thumb key (position 36)  →  NUM layer active
2. While holding, press home-row F-position key (position 16)  →  &sl 3 arms FNC sticky
3. Release NUM thumb key
4. Press leftmost left-thumb key (position 36)  →  fires &kh_toggle from FNC
```

Steps 3 and 4 are distinct presses — the sticky resolves on the next key after it's armed.
Any unintended keypress between arming and step 4 will consume the sticky layer.

## Remaining Hypotheses (lower priority now)

### 1. ZMK Studio display interference
The dongle build has `CONFIG_ZMK_STUDIO=y`. ZMK Studio adds USB/BLE protocol overhead. If it processes display events in a way that re-renders the status screen, it could override `lv_scr_load(hist)`. Scenario D above would confirm this.
**How to check**: Build once without Studio (`cmake-args: -DCONFIG_ZMK_STUDIO=n`) and test.

### 2. FNC sticky layer consumed early (hypothesis #3 from original doc)
If the `&sl 3` sticky resolves prematurely (e.g., due to a bounce or inadvertent keypress), the FNC layer never activates for step 4. The NUM layer test (position 40) directly bypasses this.

## Key Files for Reference

- Behavior drivers: [`boards/shields/prospector_adapter/src/behaviors/key_history_behaviors.c`](../prospector-zmk-module/boards/shields/prospector_adapter/src/behaviors/key_history_behaviors.c)
- Screen: [`boards/shields/prospector_adapter/src/widgets/key_history_screen.c`](../prospector-zmk-module/boards/shields/prospector_adapter/src/widgets/key_history_screen.c)
- Listener + ring buffer: [`boards/shields/prospector_adapter/src/key_history_listener.c`](../prospector-zmk-module/boards/shields/prospector_adapter/src/key_history_listener.c)
- Display init hook: [`boards/shields/prospector_adapter/src/custom_status_screen.c`](../prospector-zmk-module/boards/shields/prospector_adapter/src/custom_status_screen.c)
- Kconfig defaults: [`boards/shields/prospector_adapter/Kconfig.defconfig`](../prospector-zmk-module/boards/shields/prospector_adapter/Kconfig.defconfig)
- Keymap: [`config/toucan.keymap`](config/toucan.keymap)
- Dongle conf: [`config/toucan_dongle.conf`](config/toucan_dongle.conf)
