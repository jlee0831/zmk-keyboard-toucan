# Key History Display — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a scrollable key history screen to the Prospector dongle display, toggle-accessible via `&kh_toggle`, showing physical key positions, resolved keycodes, active layers, and layer change events.

**Architecture:** Fork `carrefinho/prospector-zmk-module`, add a ZMK event listener that maintains a 64-entry ring buffer, an LVGL history screen that renders it (offset-based scroll, full rebuild on each scroll), and three ZMK behaviors (toggle, scroll up, scroll down). The `toucan-zmk` config repo updates `west.yml` to point at the fork and adds three keymap bindings.

**Tech Stack:** C, ZMK v0.3, Zephyr RTOS, LVGL v8, nRF52840 (seeeduino_xiao_ble), ST7789 280×240 landscape RGB565

> **Note on testing:** ZMK is bare-metal firmware — there is no unit test runner. Each task's verification step is a successful `west build` (no compilation errors). End-to-end behavior is verified by flashing and manual testing. If you don't have a local ZMK west workspace, push to GitHub and verify CI passes (build.yaml).
>
> **Local build command** (from your ZMK west workspace `app/` directory):
> ```bash
> west build -b seeeduino_xiao_ble -p -- \
>   -DSHIELD="toucan_dongle rgbled_adapter prospector_adapter" \
>   -DZMK_CONFIG="/path/to/toucan-zmk/config" \
>   -DSNIPPET="studio-rpc-usb-uart" \
>   -DCONFIG_ZMK_STUDIO=y
> ```

---

## File Map

All fork changes are under `boards/shields/prospector_adapter/` unless noted.

| File | Action | Responsibility |
|---|---|---|
| `include/key_history.h` | Create | Shared types (`kh_entry_t`, `kh_event_type_t`) and API declarations |
| `src/key_history_listener.c` | Create | ZMK event listener; ring buffer; recording pause |
| `src/widgets/key_history_screen.c` | Create | LVGL history screen creation and rendering |
| `src/behaviors/key_history_behaviors.c` | Create | `kh_toggle`, `kh_scroll_up`, `kh_scroll_down` behavior drivers |
| `dts/bindings/zmk,key-history-toggle.yaml` | Create | DTS binding for `&kh_toggle` (module root `dts/bindings/`) |
| `dts/bindings/zmk,key-history-scroll.yaml` | Create | DTS binding for `&kh_scroll_up` / `&kh_scroll_down` |
| `Kconfig.shield` | Modify | Add `TOUCAN_KEY_HISTORY` Kconfig symbol |
| `Kconfig.defconfig` | Modify | Enable Montserrat 12 and 14 fonts when feature is on |
| `CMakeLists.txt` | Modify | Compile new sources when `CONFIG_TOUCAN_KEY_HISTORY=y` |
| `src/custom_status_screen.c` | Modify | Init history screen + listener at startup |
| `zephyr/module.yml` (module root) | Modify | Add `dts_root: .` so binding YAMLs are discovered |
| `config/west.yml` (toucan-zmk repo) | Modify | Point `prospector-zmk-module` at fork |
| `config/toucan.conf` (toucan-zmk repo) | Modify | Add `CONFIG_TOUCAN_KEY_HISTORY=y` |
| `config/toucan.keymap` (toucan-zmk repo) | Modify | Add behavior instances and three bindings |

---

## Task 1: Fork setup and west.yml

**Files:**
- Modify: `config/west.yml`

- [ ] **Step 1: Fork the prospector-zmk-module repo**

  Go to https://github.com/carrefinho/prospector-zmk-module and click Fork. Name your fork `prospector-zmk-module` under your own GitHub account (e.g., `yourgithubuser/prospector-zmk-module`). Clone it locally alongside your toucan-zmk workspace.

- [ ] **Step 2: Update west.yml to point at your fork**

  In `config/west.yml`, replace the `prospector-zmk-module` entry:

  ```yaml
  # Before:
  - name: prospector-zmk-module
    remote: carrefinho
    revision: main

  # After:
  - name: prospector-zmk-module
    remote: yourgithubuser          # add a new remote entry, or reuse an existing one
    revision: main
  ```

  If you don't have a remote entry for your GitHub account yet, add it to the `remotes:` list:
  ```yaml
  remotes:
    - name: yourgithubuser
      url-base: https://github.com/yourgithubuser
  ```

- [ ] **Step 3: Verify the existing dongle firmware still builds**

  Run the local build command or push to GitHub and confirm CI passes. Expected: dongle firmware builds with no changes to behavior.

- [ ] **Step 4: Commit**

  ```bash
  cd /path/to/toucan-zmk
  git add config/west.yml
  git commit -m "feat: point prospector-zmk-module at personal fork"
  ```

---

## Task 2: Module infrastructure

**Files:**
- Modify: `zephyr/module.yml` (fork root)
- Modify: `boards/shields/prospector_adapter/Kconfig.shield`
- Modify: `boards/shields/prospector_adapter/Kconfig.defconfig`
- Modify: `boards/shields/prospector_adapter/CMakeLists.txt`
- Create: `boards/shields/prospector_adapter/include/key_history.h` (stub)
- Create: `boards/shields/prospector_adapter/src/key_history_listener.c` (stub)
- Create: `boards/shields/prospector_adapter/src/widgets/key_history_screen.c` (stub)
- Create: `boards/shields/prospector_adapter/src/behaviors/key_history_behaviors.c` (stub)
- Create: `dts/bindings/zmk,key-history-toggle.yaml` (fork root)
- Create: `dts/bindings/zmk,key-history-scroll.yaml` (fork root)

- [ ] **Step 1: Add `dts_root` to `zephyr/module.yml`**

  ```yaml
  name: prospector-zmk-module
  build:
    cmake: .
    kconfig: Kconfig
    settings:
      board_root: .
      dts_root: .
    depends:
      - lvgl
  ```

- [ ] **Step 2: Add `TOUCAN_KEY_HISTORY` to `Kconfig.shield`**

  Append to the existing file (which currently only has `config SHIELD_PROSPECTOR_ADAPTER`):

  ```kconfig
  config SHIELD_PROSPECTOR_ADAPTER
      def_bool $(shields_list_contains,prospector_adapter)

  config TOUCAN_KEY_HISTORY
      bool "Key history display widget"
      depends on SHIELD_PROSPECTOR_ADAPTER
      default n
  ```

- [ ] **Step 3: Add font defaults to `Kconfig.defconfig`**

  Append to the existing file after the last existing block:

  ```kconfig
  if TOUCAN_KEY_HISTORY

  config LV_FONT_MONTSERRAT_12
      default y

  config LV_FONT_MONTSERRAT_14
      default y

  endif # TOUCAN_KEY_HISTORY
  ```

- [ ] **Step 4: Add new sources to `CMakeLists.txt`**

  Append inside the existing `if(CONFIG_SHIELD_PROSPECTOR_ADAPTER)` block, after the existing `zephyr_library_sources(...)` calls:

  ```cmake
  if(CONFIG_TOUCAN_KEY_HISTORY)
    zephyr_library_include_directories(include)
    zephyr_library_sources(
      src/key_history_listener.c
      src/widgets/key_history_screen.c
      src/behaviors/key_history_behaviors.c
    )
  endif()
  ```

- [ ] **Step 5: Create stub header `include/key_history.h`**

  ```c
  #pragma once
  #include <zephyr/kernel.h>
  #include <lvgl.h>

  /* stub — filled in Task 3 */
  void kh_stub(void);
  ```

- [ ] **Step 6: Create stub `src/key_history_listener.c`**

  ```c
  #include "key_history.h"

  void kh_stub(void) {}
  ```

- [ ] **Step 7: Create stub `src/widgets/key_history_screen.c`**

  ```c
  #include "key_history.h"
  ```

- [ ] **Step 8: Create stub `src/behaviors/key_history_behaviors.c`**

  ```c
  #include "key_history.h"
  ```

- [ ] **Step 9: Create `dts/bindings/zmk,key-history-toggle.yaml`** (at fork root, not inside `boards/`)

  ```yaml
  description: Key history toggle behavior

  compatible: "zmk,key-history-toggle"

  include:
    - name: zero_param.yaml
  ```

- [ ] **Step 10: Create `dts/bindings/zmk,key-history-scroll.yaml`** (at fork root)

  ```yaml
  description: Key history scroll behavior

  compatible: "zmk,key-history-scroll"

  include:
    - name: zero_param.yaml

  properties:
    direction:
      type: int
      required: true
      description: "0 = scroll up toward older entries, 1 = scroll down toward newer entries"
  ```

- [ ] **Step 11: Build with feature disabled (no change)**

  In `config/toucan.conf`, do NOT add `CONFIG_TOUCAN_KEY_HISTORY=y` yet. Build the dongle firmware. Expected: builds clean, zero changes to behavior.

- [ ] **Step 12: Build with feature enabled**

  Add to `config/toucan.conf` temporarily:
  ```
  CONFIG_TOUCAN_KEY_HISTORY=y
  ```
  Build the dongle firmware. Expected: builds clean (stubs compile, nothing linked yet).

- [ ] **Step 13: Commit to fork**

  ```bash
  cd /path/to/prospector-zmk-module-fork
  git add .
  git commit -m "feat: scaffold key history feature (stubs)"
  git push
  ```

---

## Task 3: Shared types and ring buffer

**Files:**
- Modify: `boards/shields/prospector_adapter/include/key_history.h` (replace stub)
- Modify: `boards/shields/prospector_adapter/src/key_history_listener.c` (replace stub)

- [ ] **Step 1: Write `include/key_history.h`**

  ```c
  #pragma once
  #include <zephyr/kernel.h>
  #include <stdint.h>
  #include <stdbool.h>
  #include <lvgl.h>

  #define KH_RING_BUFFER_SIZE 64

  typedef enum {
      KH_KEY_PRESS,
      KH_KEY_RELEASE,
      KH_LAYER_CHANGE,
  } kh_event_type_t;

  typedef struct {
      kh_event_type_t type;
      uint8_t  layer;        /* active layer at time of event */
      uint32_t position;     /* physical key position (0xFFFFFFFF = unknown) */
      uint32_t keycode;      /* resolved HID keycode (0 = none yet) */
      uint8_t  mods;         /* explicit modifiers at time of press */
      uint8_t  new_layer;    /* layer that changed (KH_LAYER_CHANGE only) */
      bool     layer_active; /* true=activated, false=deactivated (KH_LAYER_CHANGE only) */
      uint32_t timestamp_ms; /* k_uptime_get() truncated to 32-bit */
  } kh_entry_t;

  /* Ring buffer API — callable from any thread (mutex-protected) */
  void     kh_push(const kh_entry_t *entry);
  uint8_t  kh_count(void);
  /* index 0 = newest entry, index 1 = second newest, etc. */
  bool     kh_get(uint8_t newest_offset, kh_entry_t *out);
  /* Attach keycode to most recent unresolved KH_KEY_PRESS.
   * If none exists, stores as standalone KH_KEY_PRESS with position=0xFFFFFFFF. */
  void     kh_attach_keycode(uint32_t keycode, uint8_t mods);

  /* Recording pause — called from behavior/display thread */
  void kh_set_recording(bool enabled);
  bool kh_is_recording(void);

  /* Screen state — accessed from both threads; atomic */
  void kh_set_active(bool active);
  bool kh_is_active(void);

  /* Scroll offset — only accessed from display thread */
  void    kh_set_scroll_offset(uint8_t offset);
  uint8_t kh_get_scroll_offset(void);

  /* Listener init — called once at startup */
  void kh_listener_init(void);

  /* Screen API — called from display thread only */
  lv_obj_t *kh_screen_create(void);
  void      kh_screen_rebuild(void);

  /* Behavior API — called from ZMK main thread, dispatches to display thread */
  void kh_cmd_toggle(void);
  void kh_cmd_scroll_up(void);
  void kh_cmd_scroll_down(void);
  ```

- [ ] **Step 2: Write `src/key_history_listener.c`**

  ```c
  #include <zephyr/kernel.h>
  #include <zmk/events/position_state_changed.h>
  #include <zmk/events/layer_state_changed.h>
  #include <zmk/events/keycode_state_changed.h>
  #include <zmk/event_manager.h>
  #include <zmk/keymap.h>
  #include "key_history.h"

  /* ── Ring buffer ─────────────────────────────────────────── */

  static kh_entry_t kh_buf[KH_RING_BUFFER_SIZE];
  static uint8_t    kh_head  = 0; /* next write slot */
  static uint8_t    kh_cnt   = 0; /* number of valid entries */
  K_MUTEX_DEFINE(kh_mutex);

  void kh_push(const kh_entry_t *entry) {
      k_mutex_lock(&kh_mutex, K_FOREVER);
      kh_buf[kh_head] = *entry;
      kh_head = (kh_head + 1) % KH_RING_BUFFER_SIZE;
      if (kh_cnt < KH_RING_BUFFER_SIZE) {
          kh_cnt++;
      }
      k_mutex_unlock(&kh_mutex);
  }

  uint8_t kh_count(void) {
      k_mutex_lock(&kh_mutex, K_FOREVER);
      uint8_t c = kh_cnt;
      k_mutex_unlock(&kh_mutex);
      return c;
  }

  bool kh_get(uint8_t newest_offset, kh_entry_t *out) {
      k_mutex_lock(&kh_mutex, K_FOREVER);
      if (newest_offset >= kh_cnt) {
          k_mutex_unlock(&kh_mutex);
          return false;
      }
      /* newest is at (kh_head - 1), offset 0 steps back from there */
      uint8_t idx = (kh_head - 1 - newest_offset + KH_RING_BUFFER_SIZE * 2)
                    % KH_RING_BUFFER_SIZE;
      *out = kh_buf[idx];
      k_mutex_unlock(&kh_mutex);
      return true;
  }

  void kh_attach_keycode(uint32_t keycode, uint8_t mods) {
      k_mutex_lock(&kh_mutex, K_FOREVER);
      /* Walk backward from newest looking for an unresolved KH_KEY_PRESS */
      for (uint8_t i = 0; i < kh_cnt && i < 4; i++) {
          uint8_t idx = (kh_head - 1 - i + KH_RING_BUFFER_SIZE * 2) % KH_RING_BUFFER_SIZE;
          if (kh_buf[idx].type == KH_KEY_PRESS && kh_buf[idx].keycode == 0) {
              kh_buf[idx].keycode = keycode;
              kh_buf[idx].mods    = mods;
              k_mutex_unlock(&kh_mutex);
              return;
          }
      }
      /* No unresolved press — store standalone */
      k_mutex_unlock(&kh_mutex);
      kh_entry_t e = {
          .type         = KH_KEY_PRESS,
          .layer        = zmk_keymap_highest_layer_active(),
          .position     = 0xFFFFFFFF,
          .keycode      = keycode,
          .mods         = mods,
          .timestamp_ms = (uint32_t)k_uptime_get(),
      };
      kh_push(&e);
  }

  /* ── Recording / active flags ────────────────────────────── */

  static atomic_t kh_recording = ATOMIC_INIT(1);
  static atomic_t kh_active    = ATOMIC_INIT(0);
  static uint8_t  kh_scroll    = 0;

  void kh_set_recording(bool en) { atomic_set(&kh_recording, en ? 1 : 0); }
  bool kh_is_recording(void)     { return atomic_get(&kh_recording) != 0; }
  void kh_set_active(bool a)     { atomic_set(&kh_active, a ? 1 : 0); }
  bool kh_is_active(void)        { return atomic_get(&kh_active) != 0; }
  void kh_set_scroll_offset(uint8_t o) { kh_scroll = o; }
  uint8_t kh_get_scroll_offset(void)   { return kh_scroll; }

  /* ── Event listener ─────────────────────────────────────── */

  static int kh_event_handler(const zmk_event_t *eh) {
      if (!kh_is_recording()) {
          return ZMK_EV_EVENT_BUBBLE;
      }

      const struct zmk_position_state_changed *pos =
          as_zmk_position_state_changed(eh);
      if (pos) {
          kh_entry_t e = {
              .type         = pos->state ? KH_KEY_PRESS : KH_KEY_RELEASE,
              .layer        = zmk_keymap_highest_layer_active(),
              .position     = pos->position,
              .keycode      = 0,
              .timestamp_ms = (uint32_t)k_uptime_get(),
          };
          kh_push(&e);
          return ZMK_EV_EVENT_BUBBLE;
      }

      const struct zmk_layer_state_changed *layer =
          as_zmk_layer_state_changed(eh);
      if (layer) {
          kh_entry_t e = {
              .type         = KH_LAYER_CHANGE,
              .layer        = zmk_keymap_highest_layer_active(),
              .new_layer    = layer->layer,
              .layer_active = layer->state,
              .timestamp_ms = (uint32_t)k_uptime_get(),
          };
          kh_push(&e);
          return ZMK_EV_EVENT_BUBBLE;
      }

      const struct zmk_keycode_state_changed *kc =
          as_zmk_keycode_state_changed(eh);
      if (kc && kc->state) {
          kh_attach_keycode(kc->keycode, kc->explicit_modifiers);
      }

      return ZMK_EV_EVENT_BUBBLE;
  }

  ZMK_LISTENER(key_history, kh_event_handler);
  ZMK_SUBSCRIPTION(key_history, zmk_position_state_changed);
  ZMK_SUBSCRIPTION(key_history, zmk_layer_state_changed);
  ZMK_SUBSCRIPTION(key_history, zmk_keycode_state_changed);

  void kh_listener_init(void) {
      /* Nothing to do — ZMK_LISTENER/ZMK_SUBSCRIPTION handle registration */
  }
  ```

- [ ] **Step 3: Build with `CONFIG_TOUCAN_KEY_HISTORY=y`**

  Expected: compiles clean. If `zmk_keymap_highest_layer_active` is not found, check the correct function name in ZMK v0.3's `app/include/zmk/keymap.h` — it may be `zmk_keymap_layer_state_is_active` or similar. Use the highest-numbered active layer.

- [ ] **Step 4: Commit to fork**

  ```bash
  git add boards/shields/prospector_adapter/include/key_history.h \
          boards/shields/prospector_adapter/src/key_history_listener.c
  git commit -m "feat: add ring buffer and ZMK event listener for key history"
  ```

---

## Task 4: LVGL history screen

**Files:**
- Modify: `boards/shields/prospector_adapter/src/widgets/key_history_screen.c` (replace stub)

- [ ] **Step 1: Write `src/widgets/key_history_screen.c`**

  ```c
  #include <zephyr/kernel.h>
  #include <lvgl.h>
  #include <string.h>
  #include <stdio.h>
  #include "key_history.h"

  #define KH_SCREEN_W      280
  #define KH_SCREEN_H      240
  #define KH_HEADER_H       32
  #define KH_LIST_H        (KH_SCREEN_H - KH_HEADER_H)
  #define KH_ROW_H          20
  #define KH_VISIBLE_ROWS  (KH_LIST_H / KH_ROW_H)  /* 10 */

  /* Column x positions and widths */
  #define KH_COL_IDX_X      2
  #define KH_COL_IDX_W     16
  #define KH_COL_POS_X     20
  #define KH_COL_POS_W     32
  #define KH_COL_ARR_X     54
  #define KH_COL_ARR_W     14
  #define KH_COL_KC_X      70
  #define KH_COL_KC_W      96
  #define KH_COL_BADGE_X  168
  #define KH_COL_BADGE_W   64
  #define KH_COL_TIME_X   234
  #define KH_COL_TIME_W    44

  static lv_obj_t *s_screen    = NULL;
  static lv_obj_t *s_list_cont = NULL;

  /* ── Layer badge colors ───────────────────────────────────── */

  typedef struct { uint32_t bg; uint32_t text; } kh_layer_color_t;

  static const kh_layer_color_t kh_layer_colors[] = {
      {0x21262d, 0x8b949e}, /* BASE (0) gray   */
      {0x1f2d3d, 0x58a6ff}, /* SYM  (1) blue   */
      {0x2d1f3d, 0xbc8cff}, /* NUM  (2) purple */
      {0x2d1d0d, 0xf0883e}, /* FNC  (3) orange */
  };

  static kh_layer_color_t kh_layer_color(uint8_t layer) {
      if (layer < ARRAY_SIZE(kh_layer_colors)) {
          return kh_layer_colors[layer];
      }
      return kh_layer_colors[0];
  }

  /* ── Keycode → short string ───────────────────────────────── */

  static const char *kh_kc_str(uint32_t kc, uint8_t mods, char *buf, size_t len) {
      static const struct { uint32_t kc; const char *s; } tbl[] = {
          {0x04,"A"},{0x05,"B"},{0x06,"C"},{0x07,"D"},{0x08,"E"},
          {0x09,"F"},{0x0A,"G"},{0x0B,"H"},{0x0C,"I"},{0x0D,"J"},
          {0x0E,"K"},{0x0F,"L"},{0x10,"M"},{0x11,"N"},{0x12,"O"},
          {0x13,"P"},{0x14,"Q"},{0x15,"R"},{0x16,"S"},{0x17,"T"},
          {0x18,"U"},{0x19,"V"},{0x1A,"W"},{0x1B,"X"},{0x1C,"Y"},
          {0x1D,"Z"},
          {0x1E,"1"},{0x1F,"2"},{0x20,"3"},{0x21,"4"},{0x22,"5"},
          {0x23,"6"},{0x24,"7"},{0x25,"8"},{0x26,"9"},{0x27,"0"},
          {0x28,"ENT"},{0x29,"ESC"},{0x2A,"BSP"},{0x2B,"TAB"},
          {0x2C,"SPC"},{0x2D,"-"},{0x2E,"="},{0x2F,"["},{0x30,"]"},
          {0x31,"\\"},{0x33,";"},{0x34,"'"},{0x35,"`"},{0x36,","},
          {0x37,"."},{0x38,"/"},{0x39,"CAP"},
          {0x3A,"F1"},{0x3B,"F2"},{0x3C,"F3"},{0x3D,"F4"},
          {0x3E,"F5"},{0x3F,"F6"},{0x40,"F7"},{0x41,"F8"},
          {0x42,"F9"},{0x43,"F10"},{0x44,"F11"},{0x45,"F12"},
          {0x4C,"DEL"},{0x4F,"→"},{0x50,"←"},{0x51,"↓"},{0x52,"↑"},
          {0xE0,"LCT"},{0xE1,"LSH"},{0xE2,"LAL"},{0xE3,"LGU"},
          {0xE4,"RCT"},{0xE5,"RSH"},{0xE6,"RAL"},{0xE7,"RGU"},
      };

      char prefix[8] = "";
      if (mods & 0x02 || mods & 0x20) strncat(prefix, "S+", sizeof(prefix)-1);
      if (mods & 0x01 || mods & 0x10) strncat(prefix, "C+", sizeof(prefix)-1);
      if (mods & 0x04 || mods & 0x40) strncat(prefix, "A+", sizeof(prefix)-1);
      if (mods & 0x08 || mods & 0x80) strncat(prefix, "G+", sizeof(prefix)-1);

      for (size_t i = 0; i < ARRAY_SIZE(tbl); i++) {
          if (tbl[i].kc == kc) {
              snprintf(buf, len, "%s%s", prefix, tbl[i].s);
              return buf;
          }
      }
      snprintf(buf, len, "%s%02X", prefix, (unsigned)(kc & 0xFF));
      return buf;
  }

  /* ── Layer name strings (match keymap display-name values) ── */

  static const char *kh_layer_name(uint8_t layer) {
      static const char *names[] = {"BASE", "SYM", "NUM", "FNC"};
      if (layer < ARRAY_SIZE(names)) return names[layer];
      static char fallback[5];
      snprintf(fallback, sizeof(fallback), "L%d", layer);
      return fallback;
  }

  /* ── Age → brightness (relative to newest entry timestamp) ── */

  static lv_opa_t kh_age_opacity(uint32_t age_ms) {
      if (age_ms < 2000) return LV_OPA_100;
      if (age_ms < 4000) return LV_OPA_60;
      if (age_ms < 6000) return LV_OPA_30;
      return LV_OPA_15;
  }

  /* ── Row renderer ────────────────────────────────────────── */

  static void kh_render_row(lv_obj_t *parent, const kh_entry_t *e,
                             uint8_t row_idx, uint32_t age_ms) {
      lv_opa_t opa = kh_age_opacity(age_ms);

      lv_obj_t *row = lv_obj_create(parent);
      lv_obj_set_size(row, KH_SCREEN_W, KH_ROW_H);
      lv_obj_set_pos(row, 0, row_idx * KH_ROW_H);
      lv_obj_set_style_border_width(row, 0, 0);
      lv_obj_set_style_pad_all(row, 0, 0);
      lv_obj_set_style_radius(row, 0, 0);
      lv_obj_clear_flag(row, LV_OBJ_FLAG_SCROLLABLE);

      if (e->type == KH_LAYER_CHANGE) {
          lv_obj_set_style_bg_color(row, lv_color_hex(0x0d1a0d), 0);
      } else {
          lv_obj_set_style_bg_color(row, lv_color_black(), 0);
      }
      lv_obj_set_style_bg_opa(row, LV_OPA_100, 0);

      /* Helper: create a label in this row */
      #define ROW_LABEL(x, w, color_hex, font_ptr)              \
          ({                                                      \
              lv_obj_t *_l = lv_label_create(row);              \
              lv_obj_set_pos(_l, (x), 3);                       \
              lv_obj_set_width(_l, (w));                        \
              lv_label_set_long_mode(_l, LV_LABEL_LONG_CLIP);  \
              lv_obj_set_style_text_color(_l,                   \
                  lv_color_hex(color_hex), 0);                  \
              lv_obj_set_style_text_opa(_l, opa, 0);           \
              lv_obj_set_style_text_font(_l, (font_ptr), 0);   \
              _l;                                               \
          })

      if (e->type == KH_LAYER_CHANGE) {
          lv_obj_t *sym = ROW_LABEL(KH_COL_POS_X, KH_COL_POS_W, 0x3fb950, &lv_font_montserrat_14);
          lv_label_set_text(sym, "\xE2\x97\x86"); /* UTF-8 ◆ */

          char desc[24];
          snprintf(desc, sizeof(desc), "%s %s",
                   e->layer_active ? "act" : "deact",
                   kh_layer_name(e->new_layer));
          lv_obj_t *ev = ROW_LABEL(KH_COL_KC_X, KH_COL_KC_W + KH_COL_BADGE_W,
                                    0x3fb950, &lv_font_montserrat_14);
          lv_label_set_text(ev, desc);

          char ts[10];
          snprintf(ts, sizeof(ts), "%lus", (unsigned long)(age_ms / 1000));
          lv_obj_t *t = ROW_LABEL(KH_COL_TIME_X, KH_COL_TIME_W, 0x30363d, &lv_font_montserrat_12);
          lv_label_set_text(t, ts);
      } else {
          /* Position */
          char pos_str[8];
          if (e->position == 0xFFFFFFFF) {
              snprintf(pos_str, sizeof(pos_str), "??");
          } else {
              snprintf(pos_str, sizeof(pos_str), "#%lu", (unsigned long)e->position);
          }
          lv_obj_t *pos_lbl = ROW_LABEL(KH_COL_POS_X, KH_COL_POS_W, 0x7c6af7, &lv_font_montserrat_14);
          lv_label_set_text(pos_lbl, pos_str);

          /* Arrow */
          lv_obj_t *arr = ROW_LABEL(KH_COL_ARR_X, KH_COL_ARR_W, 0x30363d, &lv_font_montserrat_14);
          lv_label_set_text(arr, "\xE2\x86\x92"); /* UTF-8 → */

          /* Keycode */
          char kc_buf[16];
          const char *kc_str = (e->keycode == 0)
              ? "..."
              : kh_kc_str(e->keycode, e->mods, kc_buf, sizeof(kc_buf));
          lv_obj_t *kc_lbl = ROW_LABEL(KH_COL_KC_X, KH_COL_KC_W, 0xe6edf3, &lv_font_montserrat_14);
          lv_label_set_text(kc_lbl, kc_str);

          /* Layer badge */
          kh_layer_color_t lc = kh_layer_color(e->layer);
          lv_obj_t *badge_bg = lv_obj_create(row);
          lv_obj_set_pos(badge_bg, KH_COL_BADGE_X, 3);
          lv_obj_set_size(badge_bg, KH_COL_BADGE_W, KH_ROW_H - 6);
          lv_obj_set_style_bg_color(badge_bg, lv_color_hex(lc.bg), 0);
          lv_obj_set_style_bg_opa(badge_bg, opa, 0);
          lv_obj_set_style_border_width(badge_bg, 0, 0);
          lv_obj_set_style_radius(badge_bg, 2, 0);
          lv_obj_set_style_pad_all(badge_bg, 0, 0);
          lv_obj_clear_flag(badge_bg, LV_OBJ_FLAG_SCROLLABLE);
          lv_obj_t *badge_lbl = lv_label_create(badge_bg);
          lv_label_set_text(badge_lbl, kh_layer_name(e->layer));
          lv_obj_set_style_text_color(badge_lbl, lv_color_hex(lc.text), 0);
          lv_obj_set_style_text_opa(badge_lbl, opa, 0);
          lv_obj_set_style_text_font(badge_lbl, &lv_font_montserrat_12, 0);
          lv_obj_center(badge_lbl);

          /* Timestamp */
          char ts[10];
          snprintf(ts, sizeof(ts), "%lus", (unsigned long)(age_ms / 1000));
          lv_obj_t *t = ROW_LABEL(KH_COL_TIME_X, KH_COL_TIME_W, 0x30363d, &lv_font_montserrat_12);
          lv_label_set_text(t, ts);
      }

      #undef ROW_LABEL
  }

  /* ── Screen create / rebuild ────────────────────────────── */

  lv_obj_t *kh_screen_create(void) {
      s_screen = lv_obj_create(NULL);
      lv_obj_set_size(s_screen, KH_SCREEN_W, KH_SCREEN_H);
      lv_obj_set_style_bg_color(s_screen, lv_color_black(), 0);
      lv_obj_set_style_bg_opa(s_screen, LV_OPA_100, 0);
      lv_obj_set_style_border_width(s_screen, 0, 0);
      lv_obj_set_style_pad_all(s_screen, 0, 0);
      lv_obj_clear_flag(s_screen, LV_OBJ_FLAG_SCROLLABLE);

      /* Header */
      lv_obj_t *header = lv_obj_create(s_screen);
      lv_obj_set_size(header, KH_SCREEN_W, KH_HEADER_H);
      lv_obj_set_pos(header, 0, 0);
      lv_obj_set_style_bg_color(header, lv_color_hex(0x0d1117), 0);
      lv_obj_set_style_bg_opa(header, LV_OPA_100, 0);
      lv_obj_set_style_border_width(header, 0, 0);
      lv_obj_set_style_pad_all(header, 0, 0);
      lv_obj_clear_flag(header, LV_OBJ_FLAG_SCROLLABLE);

      lv_obj_t *title = lv_label_create(header);
      lv_label_set_text(title, "KEY HISTORY");
      lv_obj_set_style_text_color(title, lv_color_white(), 0);
      lv_obj_set_style_text_font(title, &lv_font_montserrat_14, 0);
      lv_obj_set_pos(title, 8, 8);

      lv_obj_t *hint = lv_label_create(header);
      lv_label_set_text(hint, "j/k=scroll  H=close");
      lv_obj_set_style_text_color(hint, lv_color_hex(0x484f58), 0);
      lv_obj_set_style_text_font(hint, &lv_font_montserrat_12, 0);
      lv_obj_align(hint, LV_ALIGN_RIGHT_MID, -6, 0);

      /* List container */
      s_list_cont = lv_obj_create(s_screen);
      lv_obj_set_size(s_list_cont, KH_SCREEN_W, KH_LIST_H);
      lv_obj_set_pos(s_list_cont, 0, KH_HEADER_H);
      lv_obj_set_style_bg_color(s_list_cont, lv_color_black(), 0);
      lv_obj_set_style_bg_opa(s_list_cont, LV_OPA_100, 0);
      lv_obj_set_style_border_width(s_list_cont, 0, 0);
      lv_obj_set_style_pad_all(s_list_cont, 0, 0);
      lv_obj_clear_flag(s_list_cont, LV_OBJ_FLAG_SCROLLABLE);

      return s_screen;
  }

  void kh_screen_rebuild(void) {
      if (!s_list_cont) return;
      lv_obj_clean(s_list_cont);

      uint8_t count = kh_count();
      uint8_t offset = kh_get_scroll_offset();

      /* Find the newest entry timestamp for age calculation */
      kh_entry_t newest;
      uint32_t newest_ts = 0;
      if (kh_get(0, &newest)) {
          newest_ts = newest.timestamp_ms;
      }

      uint8_t row = 0;
      uint8_t skip = offset; /* skip this many from newest (scroll position) */
      uint8_t src  = 0;      /* index into ring buffer (0=newest) */

      while (row < KH_VISIBLE_ROWS && src < count) {
          kh_entry_t e;
          if (!kh_get(src, &e)) break;
          src++;

          /* Skip key releases — not displayed */
          if (e.type == KH_KEY_RELEASE) continue;

          if (skip > 0) {
              skip--;
              continue;
          }

          uint32_t age_ms = (newest_ts >= e.timestamp_ms)
                            ? (newest_ts - e.timestamp_ms)
                            : 0;
          kh_render_row(s_list_cont, &e, row, age_ms);
          row++;
      }
  }
  ```

- [ ] **Step 2: Build with `CONFIG_TOUCAN_KEY_HISTORY=y`**

  Expected: compiles clean. If font symbols like `lv_font_montserrat_14` are not found, verify `CONFIG_LV_FONT_MONTSERRAT_14=y` is being applied — check the build's `.config` file.

- [ ] **Step 3: Commit to fork**

  ```bash
  git add boards/shields/prospector_adapter/src/widgets/key_history_screen.c
  git commit -m "feat: add LVGL key history screen with offset-based scroll"
  ```

---

## Task 5: ZMK behaviors

**Files:**
- Modify: `boards/shields/prospector_adapter/src/behaviors/key_history_behaviors.c` (replace stub)

The behaviors dispatch to the display thread via `k_work`. The toggle and scroll functions that actually call LVGL are run on the display thread; the behavior handlers only schedule the work.

- [ ] **Step 1: Write `src/behaviors/key_history_behaviors.c`**

  ```c
  #include <zephyr/kernel.h>
  #include <zephyr/device.h>
  #include <drivers/behavior.h>
  #include <zmk/behavior.h>
  #include <zmk/display.h>
  #include <lvgl.h>
  #include "key_history.h"

  /* ── Display-thread work items ───────────────────────────── */

  static void do_toggle(struct k_work *work) {
      bool now_active = !kh_is_active();
      kh_set_active(now_active);
      kh_set_recording(!now_active);

      if (now_active) {
          kh_set_scroll_offset(0);
          kh_screen_rebuild();
          lv_scr_load(kh_get_screen());
      } else {
          lv_scr_load(kh_get_normal_screen());
      }
  }
  K_WORK_DEFINE(toggle_work, do_toggle);

  static void do_scroll_up(struct k_work *work) {
      uint8_t count = kh_count();
      uint8_t max_offset = (count > KH_VISIBLE_ROWS) ? (count - KH_VISIBLE_ROWS) : 0;
      uint8_t cur = kh_get_scroll_offset();
      if (cur < max_offset) {
          kh_set_scroll_offset(cur + 1);
          kh_screen_rebuild();
      }
  }
  K_WORK_DEFINE(scroll_up_work, do_scroll_up);

  static void do_scroll_down(struct k_work *work) {
      uint8_t cur = kh_get_scroll_offset();
      if (cur > 0) {
          kh_set_scroll_offset(cur - 1);
          kh_screen_rebuild();
      }
  }
  K_WORK_DEFINE(scroll_down_work, do_scroll_down);

  /* ── Behavior API (called from ZMK main thread) ─────────── */

  void kh_cmd_toggle(void) {
      k_work_submit_to_queue(zmk_display_work_q(), &toggle_work);
  }
  void kh_cmd_scroll_up(void) {
      if (kh_is_active()) {
          k_work_submit_to_queue(zmk_display_work_q(), &scroll_up_work);
      }
  }
  void kh_cmd_scroll_down(void) {
      if (kh_is_active()) {
          k_work_submit_to_queue(zmk_display_work_q(), &scroll_down_work);
      }
  }

  /* ── Toggle behavior driver ─────────────────────────────── */

  static int kh_toggle_pressed(struct zmk_behavior_binding *binding,
                                struct zmk_behavior_binding_event event) {
      kh_cmd_toggle();
      return ZMK_BEHAVIOR_OPAQUE;
  }
  static int kh_toggle_released(struct zmk_behavior_binding *binding,
                                 struct zmk_behavior_binding_event event) {
      return ZMK_BEHAVIOR_OPAQUE;
  }
  static const struct behavior_driver_api kh_toggle_driver_api = {
      .binding_pressed  = kh_toggle_pressed,
      .binding_released = kh_toggle_released,
  };

  #define DT_DRV_COMPAT zmk_key_history_toggle
  DT_INST_FOREACH_STATUS_OKAY_VARGS(DEVICE_DT_INST_DEFINE,
      NULL, NULL, NULL, NULL,
      APPLICATION, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT,
      &kh_toggle_driver_api)
  #undef DT_DRV_COMPAT

  /* ── Scroll behavior driver ─────────────────────────────── */

  struct kh_scroll_cfg { int direction; };

  static int kh_scroll_pressed(struct zmk_behavior_binding *binding,
                                struct zmk_behavior_binding_event event) {
      const struct device *dev = device_get_binding(binding->behavior_dev);
      const struct kh_scroll_cfg *cfg = dev->config;
      if (cfg->direction == 0) {
          kh_cmd_scroll_up();
      } else {
          kh_cmd_scroll_down();
      }
      return ZMK_BEHAVIOR_OPAQUE;
  }
  static int kh_scroll_released(struct zmk_behavior_binding *binding,
                                  struct zmk_behavior_binding_event event) {
      return ZMK_BEHAVIOR_OPAQUE;
  }
  static const struct behavior_driver_api kh_scroll_driver_api = {
      .binding_pressed  = kh_scroll_pressed,
      .binding_released = kh_scroll_released,
  };

  #define DT_DRV_COMPAT zmk_key_history_scroll
  #define KH_SCROLL_INST(n)                                       \
      static const struct kh_scroll_cfg kh_scroll_cfg_##n = {    \
          .direction = DT_INST_PROP(n, direction),                \
      };                                                           \
      DEVICE_DT_INST_DEFINE(n, NULL, NULL, NULL,                  \
          &kh_scroll_cfg_##n,                                     \
          APPLICATION, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT,       \
          &kh_scroll_driver_api);
  DT_INST_FOREACH_STATUS_OKAY(KH_SCROLL_INST)
  #undef DT_DRV_COMPAT
  ```

- [ ] **Step 2: Add `kh_get_screen()` and `kh_get_normal_screen()` to the header**

  Append to `include/key_history.h`:
  ```c
  /* Screen references — set by custom_status_screen.c at init */
  void      kh_set_screens(lv_obj_t *normal, lv_obj_t *history);
  lv_obj_t *kh_get_screen(void);
  lv_obj_t *kh_get_normal_screen(void);
  ```

- [ ] **Step 3: Add screen reference storage to `src/key_history_listener.c`**

  Append at the bottom of `key_history_listener.c`:
  ```c
  static lv_obj_t *s_normal_screen  = NULL;
  static lv_obj_t *s_history_screen = NULL;

  void kh_set_screens(lv_obj_t *normal, lv_obj_t *history) {
      s_normal_screen  = normal;
      s_history_screen = history;
  }
  lv_obj_t *kh_get_screen(void)        { return s_history_screen; }
  lv_obj_t *kh_get_normal_screen(void) { return s_normal_screen; }
  ```

  > **Note on `zmk_display_work_q()`:** This function is declared in `zmk/display.h` and available when `CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y` (which the Prospector uses). If the linker cannot find it, check `app/include/zmk/display.h` in your ZMK west workspace for the exact declaration.

- [ ] **Step 4: Build with `CONFIG_TOUCAN_KEY_HISTORY=y`**

  Expected: compiles clean. The two DTS-based drivers won't instantiate yet (no DTS nodes exist), but the code must compile.

  > **Note on `DT_INST_FOREACH_STATUS_OKAY_VARGS`:** If this macro doesn't exist in ZMK v0.3, replace the toggle driver registration with:
  > ```c
  > #define KH_TOGGLE_INST(n) \
  >     DEVICE_DT_INST_DEFINE(n, NULL, NULL, NULL, NULL, \
  >         APPLICATION, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT, \
  >         &kh_toggle_driver_api);
  > DT_INST_FOREACH_STATUS_OKAY(KH_TOGGLE_INST)
  > ```

- [ ] **Step 5: Commit to fork**

  ```bash
  git add boards/shields/prospector_adapter/src/behaviors/key_history_behaviors.c \
          boards/shields/prospector_adapter/include/key_history.h \
          boards/shields/prospector_adapter/src/key_history_listener.c
  git commit -m "feat: add toggle and scroll ZMK behaviors for key history"
  ```

---

## Task 6: Wire up in custom_status_screen.c

**Files:**
- Modify: `boards/shields/prospector_adapter/src/custom_status_screen.c`

- [ ] **Step 1: Add the key history init block**

  In `custom_status_screen.c`, find the `zmk_display_status_screen()` function. After the line that assigns `screen` (the normal screen object), add:

  ```c
  #if IS_ENABLED(CONFIG_TOUCAN_KEY_HISTORY)
  #include "key_history.h"
  #endif
  ```

  at the top of the file with the other includes. Then inside `zmk_display_status_screen()`, after `lv_obj_t *screen = lv_obj_create(NULL);` and before `return screen;`, add:

  ```c
  #if IS_ENABLED(CONFIG_TOUCAN_KEY_HISTORY)
      lv_obj_t *hist = kh_screen_create();
      kh_set_screens(screen, hist);
      kh_listener_init();
  #endif
  ```

  The full function should look like:
  ```c
  lv_obj_t *zmk_display_status_screen() {
      lv_obj_t *screen = lv_obj_create(NULL);
      lv_obj_set_style_bg_color(screen, lv_color_black(), LV_STATE_DEFAULT);
      lv_obj_set_style_bg_opa(screen, LV_OPA_COVER, LV_STATE_DEFAULT);

      /* ... existing widget creation (battery bar, layer roller, etc.) ... */

  #if IS_ENABLED(CONFIG_TOUCAN_KEY_HISTORY)
      lv_obj_t *hist = kh_screen_create();
      kh_set_screens(screen, hist);
      kh_listener_init();
  #endif

      return screen;
  }
  ```

- [ ] **Step 2: Build with `CONFIG_TOUCAN_KEY_HISTORY=y`**

  Expected: compiles and links clean. The history screen is created at startup alongside the normal screen. No behavior change yet — toggle behaviors have no DTS nodes.

- [ ] **Step 3: Commit to fork**

  ```bash
  git add boards/shields/prospector_adapter/src/custom_status_screen.c
  git commit -m "feat: init key history screen at startup when TOUCAN_KEY_HISTORY=y"
  git push
  ```

---

## Task 7: Keymap bindings

**Files:**
- Modify: `config/toucan.conf` (toucan-zmk repo)
- Modify: `config/toucan.keymap` (toucan-zmk repo)

- [ ] **Step 1: Enable the feature in `config/toucan.conf`**

  Add to `config/toucan.conf`:
  ```
  CONFIG_TOUCAN_KEY_HISTORY=y
  ```

- [ ] **Step 2: Add behavior instances to `config/toucan.keymap`**

  In the `behaviors { ... }` block, after the last existing behavior, add:

  ```dts
  kh_toggle: kh_toggle {
      compatible = "zmk,key-history-toggle";
      #binding-cells = <0>;
  };
  kh_scroll_up: kh_scroll_up {
      compatible = "zmk,key-history-scroll";
      #binding-cells = <0>;
      direction = <0>;
  };
  kh_scroll_down: kh_scroll_down {
      compatible = "zmk,key-history-scroll";
      #binding-cells = <0>;
      direction = <1>;
  };
  ```

- [ ] **Step 3: Add bindings to the FNC layer**

  In the `fnc` layer's `bindings`, replace three of the bottom-row `&trans` entries with the history behaviors. The FNC layer bottom row is currently:
  ```
  &trans  &trans  &trans  &trans  &trans  &trans
  ```

  Change it to:
  ```
  &kh_toggle  &kh_scroll_up  &kh_scroll_down  &trans  &trans  &trans
  ```

  This places the behaviors on the three leftmost bottom-row keys of the FNC layer. Adjust to your preferred positions.

- [ ] **Step 4: Build the complete dongle firmware**

  Run the local build or push to GitHub. Expected: dongle firmware builds clean with `CONFIG_TOUCAN_KEY_HISTORY=y`. Both behavior DTS nodes are now instantiated.

- [ ] **Step 5: Commit**

  ```bash
  cd /path/to/toucan-zmk
  git add config/toucan.conf config/toucan.keymap
  git commit -m "feat: add key history keymap bindings to FNC layer"
  ```

---

## Task 8: End-to-end verification

- [ ] **Step 1: Flash the dongle firmware**

  Use the UF2 file from CI (or `west flash` locally) to flash the dongle.

- [ ] **Step 2: Verify normal screen still works**

  Power cycle the dongle. Confirm the existing Prospector status screen (layer roller, battery bar) appears as before.

- [ ] **Step 3: Verify history screen opens**

  Type a few keys on the keyboard. Then press the FNC layer key + the `&kh_toggle` position. Expected: display switches to the dark "KEY HISTORY" screen.

- [ ] **Step 4: Verify ring buffer contents**

  The rows should show the physical key positions you pressed and the resolved keycodes. Layer changes (if any) should appear as green rows.

- [ ] **Step 5: Verify scroll**

  Type enough keys to fill more than 10 rows. Open history. Press `&kh_scroll_up` a few times — older entries should appear. Press `&kh_scroll_down` to scroll back. Expected: smooth offset-based navigation.

- [ ] **Step 6: Verify recording pause**

  While the history screen is open, press several keys (including `&kh_scroll_up`). Close history with `&kh_toggle`. Re-open immediately. Expected: the scroll presses do NOT appear in the ring buffer.

- [ ] **Step 7: Verify age fading**

  Type a few keys. Wait 5–10 seconds. Open history. Expected: the older entries are visibly dimmer than the recent ones.

- [ ] **Step 8: Final commit and push**

  ```bash
  cd /path/to/prospector-zmk-module-fork
  git push

  cd /path/to/toucan-zmk
  git push
  ```

---

## Notes

**`zmk_keymap_highest_layer_active()`:** Verify the exact function name in `app/include/zmk/keymap.h`. In some ZMK versions it is `zmk_keymap_layer_state_is_active(layer)` (per-layer query) with no single "highest" function. If so, implement a small helper:
```c
static uint8_t highest_active_layer(void) {
    for (int i = ZMK_KEYMAP_LAYERS_LEN - 1; i >= 0; i--) {
        if (zmk_keymap_layer_state_is_active(i)) return i;
    }
    return 0;
}
```

**`zmk_display_work_q()`:** Declared in `app/include/zmk/display.h`. Available only when `CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y`, which the Prospector uses. If unavailable, fall back to the system work queue: `k_work_submit(&toggle_work)` — but this runs on a different thread than LVGL; test carefully for rendering artifacts.

**LVGL v8 vs v9:** ZMK v0.3 uses LVGL v8.3.x. The API above is v8. If Zephyr's LVGL module has been updated to v9 in your west workspace, consult the LVGL v9 migration guide for `lv_obj_set_style_*` API differences.

**UTF-8 characters:** `◆` is `\xE2\x97\x86` and `→` is `\xE2\x86\x92` in UTF-8. Confirm `CONFIG_LV_TXT_ENC_UTF8=y` (it is the default in LVGL v8).
