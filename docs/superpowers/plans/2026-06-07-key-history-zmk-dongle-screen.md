# Key History Display — zmk-dongle-screen Port

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port the key history display feature from the abandoned `prospector-zmk-module` fork to a fork of `janpfischer/zmk-dongle-screen`, using `ZMK_DISPLAY_WIDGET_LISTENER` for event subscriptions and fixing the behavior driver registration bug that prevented toggle/scroll from working.

**Architecture:** A fork of `zmk-dongle-screen` (`jlee0831/zmk-dongle-screen`, branch `feat/key-history`) gains three new files: a shared header, a combined widget (ring buffer + three display-thread event listeners + LVGL full-screen overlay), and ZMK behavior drivers. All ring buffer access is display-thread-only (no mutex). Behaviors dispatch to the display thread via `k_work_submit_to_queue(zmk_display_work_q(), ...)`. The `toucan-zmk` config swaps `prospector-zmk-module` for the `zmk-dongle-screen` fork.

**Tech Stack:** C, ZMK v0.3, Zephyr RTOS, LVGL v8.3, nRF52840 (seeeduino_xiao_ble), ST7789V 280×240 landscape RGB565.

> **Note on testing:** ZMK is bare-metal firmware — no unit test runner exists. Each task's verification step is a successful `west build`. End-to-end behavior is verified by flashing and manual testing. If you don't have a local ZMK west workspace, push to GitHub and confirm CI passes.
>
> **Local build command** (from ZMK west workspace `app/` directory):
> ```bash
> west build -b seeeduino_xiao_ble -p -- \
>   -DSHIELD="toucan_dongle rgbled_adapter dongle_screen" \
>   -DZMK_CONFIG="/path/to/toucan-zmk/config" \
>   -DSNIPPET="studio-rpc-usb-uart" \
>   -DCONFIG_ZMK_STUDIO=y
> ```
> To test with the feature enabled before adding it to `toucan.conf`, append `-DCONFIG_DONGLE_SCREEN_KEY_HISTORY=y` to the build command.

---

## File Map

All fork changes are in `/Users/jlee/projects/personal/zmk-dongle-screen/`.
All config changes are in `/Users/jlee/projects/personal/toucan-zmk/config/`.

| File | Action | Responsibility |
|---|---|---|
| `zephyr/module.yml` | Modify | Add `dts_root: .` so binding YAMLs are discovered |
| `boards/shields/dongle_screen/Kconfig.defconfig` | Modify | Add `DONGLE_SCREEN_KEY_HISTORY` symbol + font defaults |
| `boards/shields/dongle_screen/CMakeLists.txt` | Modify | Compile new sources when feature enabled |
| `dts/bindings/zmk,key-history-toggle.yaml` | Create | DTS binding for `&kh_toggle` |
| `dts/bindings/zmk,key-history-scroll.yaml` | Create | DTS binding for `&kh_scroll_up` / `&kh_scroll_down` |
| `boards/shields/dongle_screen/include/key_history.h` | Create | Shared types and public API |
| `boards/shields/dongle_screen/src/widgets/key_history_widget.c` | Create | Ring buffer + 3 `ZMK_DISPLAY_WIDGET_LISTENER` subscriptions + LVGL screen |
| `boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c` | Create | Fixed behavior drivers (toggle + scroll) |
| `boards/shields/dongle_screen/src/custom_status_screen.c` | Modify | Init key history screen at startup |
| `config/west.yml` | Modify | Replace `prospector-zmk-module` with `zmk-dongle-screen` fork |
| `config/toucan.conf` | Modify | Replace prospector-specific options with `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y` |

---

## Task 1: Feature branch and west.yml

**Files:**
- Modify: `config/west.yml`

- [ ] **Step 1: Create feature branch in zmk-dongle-screen fork**

  ```bash
  cd /Users/jlee/projects/personal/zmk-dongle-screen
  git checkout -b feat/key-history
  ```

  Expected: switched to a new branch `feat/key-history`.

- [ ] **Step 2: Update `config/west.yml` in toucan-zmk**

  Replace the `prospector-zmk-module` project entry and remove the unused `carrefinho` remote:

  ```yaml
  manifest:
    defaults:
      revision: v0.3
    remotes:
      - name: zmkfirmware
        url-base: https://github.com/zmkfirmware
      - name: petejohanson
        url-base: https://github.com/petejohanson/
      - name: caksoylar
        url-base: https://github.com/caksoylar
      - name: geeksville
        url-base: https://github.com/geeksville
      - name: jlee0831
        url-base: https://github.com/jlee0831
    projects:
      - name: zmk
        remote: zmkfirmware
        import: app/west.yml
      - name: cirque-input-module
        remote: geeksville
        revision: toucan
      - name: zmk-rgbled-widget
        remote: caksoylar
        revision: v0.3
      - name: zmk-dongle-screen
        remote: jlee0831
        revision: feat/key-history
    self:
      path: config
  ```

- [ ] **Step 3: Verify base dongle firmware still builds**

  Run the local build command (or push and check CI). Expected: dongle firmware builds clean with existing layer/battery/WPM widgets — no key history code yet.

- [ ] **Step 4: Commit west.yml**

  ```bash
  cd /Users/jlee/projects/personal/toucan-zmk
  git add config/west.yml
  git commit -m "feat: switch dongle screen module from prospector to zmk-dongle-screen fork"
  ```

---

## Task 2: Module infrastructure

**Files:**
- Modify: `zephyr/module.yml`
- Modify: `boards/shields/dongle_screen/Kconfig.defconfig`
- Modify: `boards/shields/dongle_screen/CMakeLists.txt`
- Create: `dts/bindings/zmk,key-history-toggle.yaml`
- Create: `dts/bindings/zmk,key-history-scroll.yaml`
- Create: `boards/shields/dongle_screen/include/key_history.h` (stub)
- Create: `boards/shields/dongle_screen/src/widgets/key_history_widget.c` (stub)
- Create: `boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c` (stub)

- [ ] **Step 1: Add `dts_root` to `zephyr/module.yml`**

  Current content:
  ```yaml
  name: dongle_screen
  build:
    cmake: .
    kconfig: Kconfig
    settings:
      board_root: .
    depends:
      - lvgl
  ```

  Replace with:
  ```yaml
  name: dongle_screen
  build:
    cmake: .
    kconfig: Kconfig
    settings:
      board_root: .
      dts_root: .
    depends:
      - lvgl
  ```

- [ ] **Step 2: Add `DONGLE_SCREEN_KEY_HISTORY` to `Kconfig.defconfig`**

  Append before the closing `endif` of the `if SHIELD_DONGLE_SCREEN` block (the very end of the file):

  ```kconfig
  config DONGLE_SCREEN_KEY_HISTORY
      bool "Key history display overlay"
      default n
      help
        Adds a scrollable key history full-screen overlay toggled by a ZMK behavior.

  if DONGLE_SCREEN_KEY_HISTORY

  config LV_FONT_MONTSERRAT_12
      default y

  config LV_FONT_MONTSERRAT_14
      default y

  endif # DONGLE_SCREEN_KEY_HISTORY
  ```

- [ ] **Step 3: Create `dts/bindings/zmk,key-history-toggle.yaml`** (at repo root, not inside `boards/`)

  ```yaml
  description: Key history toggle behavior

  compatible: "zmk,key-history-toggle"

  include:
    - name: zero_param.yaml
  ```

- [ ] **Step 4: Create `dts/bindings/zmk,key-history-scroll.yaml`** (at repo root)

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

- [ ] **Step 5: Add new sources to `CMakeLists.txt`**

  Append inside the existing `if(CONFIG_SHIELD_DONGLE_SCREEN)` block, after the existing `zephyr_library_sources(...)` calls and before `endif()`:

  ```cmake
  if(CONFIG_DONGLE_SCREEN_KEY_HISTORY)
    zephyr_library_sources(src/widgets/key_history_widget.c)
    zephyr_library_sources(src/behaviors/key_history_behaviors.c)
  endif()
  ```

- [ ] **Step 6: Create stub `boards/shields/dongle_screen/include/key_history.h`**

  ```c
  #pragma once
  #include <zephyr/kernel.h>
  #include <lvgl.h>

  /* stub — filled in Task 3 */
  void kh_widget_init(void);
  ```

- [ ] **Step 7: Create stub `boards/shields/dongle_screen/src/widgets/key_history_widget.c`**

  ```c
  #include "key_history.h"

  void kh_widget_init(void) {}
  ```

- [ ] **Step 8: Create `boards/shields/dongle_screen/src/behaviors/` directory and stub file**

  ```bash
  mkdir -p /Users/jlee/projects/personal/zmk-dongle-screen/boards/shields/dongle_screen/src/behaviors
  ```

  Create `boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c`:
  ```c
  #include "key_history.h"
  ```

- [ ] **Step 9: Build with feature disabled**

  Do NOT add `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y` yet. Run the build command. Expected: builds clean — stubs are not compiled, no behavior change.

- [ ] **Step 10: Build with feature enabled**

  Add `-DCONFIG_DONGLE_SCREEN_KEY_HISTORY=y` to the build command. Expected: stubs compile clean.

- [ ] **Step 11: Commit to fork**

  ```bash
  cd /Users/jlee/projects/personal/zmk-dongle-screen
  git add zephyr/module.yml \
          boards/shields/dongle_screen/Kconfig.defconfig \
          boards/shields/dongle_screen/CMakeLists.txt \
          dts/bindings/ \
          boards/shields/dongle_screen/include/key_history.h \
          boards/shields/dongle_screen/src/widgets/key_history_widget.c \
          boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c
  git commit -m "feat: scaffold key history feature (stubs + infrastructure)"
  git push -u origin feat/key-history
  ```

---

## Task 3: key_history.h and key_history_widget.c

**Files:**
- Modify: `boards/shields/dongle_screen/include/key_history.h` (replace stub)
- Modify: `boards/shields/dongle_screen/src/widgets/key_history_widget.c` (replace stub)

- [ ] **Step 1: Write `boards/shields/dongle_screen/include/key_history.h`**

  ```c
  #pragma once
  #include <zephyr/kernel.h>
  #include <stdint.h>
  #include <stdbool.h>
  #include <lvgl.h>

  #define KH_RING_BUFFER_SIZE 64
  #define KH_VISIBLE_ROWS     10  /* (240 - 32) / 20 */

  typedef enum {
      KH_KEY_PRESS,
      KH_KEY_RELEASE,
      KH_LAYER_CHANGE,
  } kh_event_type_t;

  typedef struct {
      kh_event_type_t type;
      uint8_t  layer;        /* active layer at time of event */
      uint32_t position;     /* physical key position (0xFFFFFFFF = unknown) */
      uint32_t keycode;      /* resolved HID keycode (0 = unresolved) */
      uint8_t  mods;         /* explicit modifiers at time of press */
      uint8_t  new_layer;    /* KH_LAYER_CHANGE only */
      bool     layer_active; /* true=activated, false=deactivated (KH_LAYER_CHANGE only) */
      uint32_t timestamp_ms;
  } kh_entry_t;

  /* Ring buffer — display-thread-only, no mutex */
  uint8_t  kh_count(void);

  /* State — atomic where cross-thread, display-thread-only otherwise */
  void    kh_set_recording(bool enabled);
  bool    kh_is_recording(void);
  void    kh_set_active(bool active);
  bool    kh_is_active(void);
  void    kh_set_scroll_offset(uint8_t offset);
  uint8_t kh_get_scroll_offset(void);

  /* Screen lifecycle — called from display thread */
  lv_obj_t *kh_screen_create(void);
  void      kh_screen_rebuild(void);
  void      kh_set_screens(lv_obj_t *normal, lv_obj_t *history);
  lv_obj_t *kh_get_screen(void);
  lv_obj_t *kh_get_normal_screen(void);

  /* Widget init — registers ZMK_DISPLAY_WIDGET_LISTENER subscriptions */
  void kh_widget_init(void);

  /* Behavior API — called from ZMK main thread, dispatches to display thread */
  void kh_cmd_toggle(void);
  void kh_cmd_scroll_up(void);
  void kh_cmd_scroll_down(void);
  ```

- [ ] **Step 2: Write `boards/shields/dongle_screen/src/widgets/key_history_widget.c`**

  ```c
  #include <zephyr/kernel.h>
  #include <lvgl.h>
  #include <string.h>
  #include <stdio.h>
  #include <zmk/display.h>
  #include <zmk/events/position_state_changed.h>
  #include <zmk/events/layer_state_changed.h>
  #include <zmk/events/keycode_state_changed.h>
  #include <zmk/event_manager.h>
  #include <zmk/keymap.h>
  #include "key_history.h"

  /* ── Ring buffer — display-thread-only ──────────────────── */

  static kh_entry_t kh_buf[KH_RING_BUFFER_SIZE];
  static uint8_t    kh_head = 0;
  static uint8_t    kh_cnt  = 0;

  static void kh_push(const kh_entry_t *entry) {
      kh_buf[kh_head] = *entry;
      kh_head = (kh_head + 1) % KH_RING_BUFFER_SIZE;
      if (kh_cnt < KH_RING_BUFFER_SIZE) {
          kh_cnt++;
      }
  }

  static bool kh_get(uint8_t newest_offset, kh_entry_t *out) {
      if (newest_offset >= kh_cnt) return false;
      uint8_t idx = (kh_head - 1 - newest_offset + KH_RING_BUFFER_SIZE * 2)
                    % KH_RING_BUFFER_SIZE;
      *out = kh_buf[idx];
      return true;
  }

  uint8_t kh_count(void) { return kh_cnt; }

  /* ── State ───────────────────────────────────────────────── */

  static atomic_t kh_recording = ATOMIC_INIT(1);
  static atomic_t kh_active    = ATOMIC_INIT(0);
  static uint8_t  kh_scroll    = 0;

  void    kh_set_recording(bool en) { atomic_set(&kh_recording, en ? 1 : 0); }
  bool    kh_is_recording(void)     { return atomic_get(&kh_recording) != 0; }
  void    kh_set_active(bool a)     { atomic_set(&kh_active, a ? 1 : 0); }
  bool    kh_is_active(void)        { return atomic_get(&kh_active) != 0; }
  void    kh_set_scroll_offset(uint8_t o) { kh_scroll = o; }
  uint8_t kh_get_scroll_offset(void)      { return kh_scroll; }

  /* ── Screen references ───────────────────────────────────── */

  static lv_obj_t *s_normal_screen  = NULL;
  static lv_obj_t *s_history_screen = NULL;
  static lv_obj_t *s_list_cont      = NULL;

  void      kh_set_screens(lv_obj_t *normal, lv_obj_t *history) {
      s_normal_screen  = normal;
      s_history_screen = history;
  }
  lv_obj_t *kh_get_screen(void)        { return s_history_screen; }
  lv_obj_t *kh_get_normal_screen(void) { return s_normal_screen; }

  /* ── Layer badge colors ───────────────────────────────────── */

  typedef struct { uint32_t bg; uint32_t text; } kh_layer_color_t;

  static const kh_layer_color_t kh_layer_colors[] = {
      {0x21262d, 0x8b949e},
      {0x1f2d3d, 0x58a6ff},
      {0x2d1f3d, 0xbc8cff},
      {0x2d1d0d, 0xf0883e},
  };

  static kh_layer_color_t kh_layer_color(uint8_t layer) {
      if (layer < ARRAY_SIZE(kh_layer_colors)) return kh_layer_colors[layer];
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
          {0x4C,"DEL"},{0x4F,"\xe2\x86\x92"},{0x50,"\xe2\x86\x90"},
          {0x51,"\xe2\x86\x93"},{0x52,"\xe2\x86\x91"},
          {0xE0,"LCT"},{0xE1,"LSH"},{0xE2,"LAL"},{0xE3,"LGU"},
          {0xE4,"RCT"},{0xE5,"RSH"},{0xE6,"RAL"},{0xE7,"RGU"},
      };

      char prefix[10] = "";
      if (mods & 0x02 || mods & 0x20) strncat(prefix, "S+", sizeof(prefix) - strlen(prefix) - 1);
      if (mods & 0x01 || mods & 0x10) strncat(prefix, "C+", sizeof(prefix) - strlen(prefix) - 1);
      if (mods & 0x04 || mods & 0x40) strncat(prefix, "A+", sizeof(prefix) - strlen(prefix) - 1);
      if (mods & 0x08 || mods & 0x80) strncat(prefix, "G+", sizeof(prefix) - strlen(prefix) - 1);

      for (size_t i = 0; i < ARRAY_SIZE(tbl); i++) {
          if (tbl[i].kc == kc) {
              snprintf(buf, len, "%s%s", prefix, tbl[i].s);
              return buf;
          }
      }
      snprintf(buf, len, "%s%02X", prefix, (unsigned)(kc & 0xFF));
      return buf;
  }

  static const char *kh_layer_name(uint8_t layer) {
      static const char *names[] = {"BASE", "SYM", "NUM", "FNC"};
      if (layer < ARRAY_SIZE(names)) return names[layer];
      static char fallback[5];
      snprintf(fallback, sizeof(fallback), "L%d", layer);
      return fallback;
  }

  static lv_opa_t kh_age_opacity(uint32_t age_ms) {
      if (age_ms < 2000) return LV_OPA_100;
      if (age_ms < 4000) return LV_OPA_60;
      if (age_ms < 6000) return LV_OPA_30;
      return LV_OPA_10;
  }

  /* ── Row renderer ────────────────────────────────────────── */

  #define KH_SCREEN_W  280
  #define KH_HEADER_H   32
  #define KH_ROW_H      20
  #define KH_COL_POS_X    8
  #define KH_COL_POS_W   36
  #define KH_COL_ARR_X   46
  #define KH_COL_ARR_W   14
  #define KH_COL_KC_X    62
  #define KH_COL_KC_W    96
  #define KH_COL_BADGE_X 160
  #define KH_COL_BADGE_W  68
  #define KH_COL_TIME_X  230
  #define KH_COL_TIME_W   48

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
      lv_obj_set_style_bg_color(row,
          e->type == KH_LAYER_CHANGE ? lv_color_hex(0x0d1a0d) : lv_color_black(), 0);
      lv_obj_set_style_bg_opa(row, LV_OPA_100, 0);

  #define ROW_LABEL(x, w, color_hex, font_ptr)              \
      ({                                                     \
          lv_obj_t *_l = lv_label_create(row);             \
          lv_obj_set_pos(_l, (x), 3);                      \
          lv_obj_set_width(_l, (w));                       \
          lv_label_set_long_mode(_l, LV_LABEL_LONG_CLIP); \
          lv_obj_set_style_text_color(_l,                  \
              lv_color_hex(color_hex), 0);                 \
          lv_obj_set_style_text_opa(_l, opa, 0);          \
          lv_obj_set_style_text_font(_l, (font_ptr), 0);  \
          _l;                                              \
      })

      if (e->type == KH_LAYER_CHANGE) {
          lv_obj_t *sym = ROW_LABEL(KH_COL_POS_X, KH_COL_POS_W, 0x3fb950, &lv_font_montserrat_14);
          lv_label_set_text(sym, "\xe2\x97\x86");

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
          char pos_str[8];
          if (e->position == 0xFFFFFFFF) {
              snprintf(pos_str, sizeof(pos_str), "??");
          } else {
              snprintf(pos_str, sizeof(pos_str), "#%lu", (unsigned long)e->position);
          }
          lv_obj_t *pos_lbl = ROW_LABEL(KH_COL_POS_X, KH_COL_POS_W, 0x7c6af7, &lv_font_montserrat_14);
          lv_label_set_text(pos_lbl, pos_str);

          lv_obj_t *arr = ROW_LABEL(KH_COL_ARR_X, KH_COL_ARR_W, 0x30363d, &lv_font_montserrat_14);
          lv_label_set_text(arr, "\xe2\x86\x92");

          char kc_buf[16];
          const char *kc_str = (e->keycode == 0)
              ? "..."
              : kh_kc_str(e->keycode, e->mods, kc_buf, sizeof(kc_buf));
          lv_obj_t *kc_lbl = ROW_LABEL(KH_COL_KC_X, KH_COL_KC_W, 0xe6edf3, &lv_font_montserrat_14);
          lv_label_set_text(kc_lbl, kc_str);

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

          char ts[10];
          snprintf(ts, sizeof(ts), "%lus", (unsigned long)(age_ms / 1000));
          lv_obj_t *t = ROW_LABEL(KH_COL_TIME_X, KH_COL_TIME_W, 0x30363d, &lv_font_montserrat_12);
          lv_label_set_text(t, ts);
      }
  #undef ROW_LABEL
  }

  /* ── Screen create / rebuild ─────────────────────────────── */

  lv_obj_t *kh_screen_create(void) {
      lv_obj_t *screen = lv_obj_create(NULL);
      lv_obj_set_size(screen, KH_SCREEN_W, 240);
      lv_obj_set_style_bg_color(screen, lv_color_black(), 0);
      lv_obj_set_style_bg_opa(screen, LV_OPA_100, 0);
      lv_obj_set_style_border_width(screen, 0, 0);
      lv_obj_set_style_pad_all(screen, 0, 0);
      lv_obj_clear_flag(screen, LV_OBJ_FLAG_SCROLLABLE);

      lv_obj_t *header = lv_obj_create(screen);
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

      s_list_cont = lv_obj_create(screen);
      lv_obj_set_size(s_list_cont, KH_SCREEN_W, 240 - KH_HEADER_H);
      lv_obj_set_pos(s_list_cont, 0, KH_HEADER_H);
      lv_obj_set_style_bg_color(s_list_cont, lv_color_black(), 0);
      lv_obj_set_style_bg_opa(s_list_cont, LV_OPA_100, 0);
      lv_obj_set_style_border_width(s_list_cont, 0, 0);
      lv_obj_set_style_pad_all(s_list_cont, 0, 0);
      lv_obj_clear_flag(s_list_cont, LV_OBJ_FLAG_SCROLLABLE);

      return screen;
  }

  void kh_screen_rebuild(void) {
      if (!s_list_cont) return;
      lv_obj_clean(s_list_cont);

      uint8_t count  = kh_count();
      uint8_t offset = kh_get_scroll_offset();

      kh_entry_t newest;
      uint32_t newest_ts = 0;
      if (kh_get(0, &newest)) {
          newest_ts = newest.timestamp_ms;
      }

      uint8_t row  = 0;
      uint8_t skip = offset;
      uint8_t src  = 0;

      while (row < KH_VISIBLE_ROWS && src < count) {
          kh_entry_t e;
          if (!kh_get(src, &e)) break;
          src++;
          if (e.type == KH_KEY_RELEASE) continue;
          if (skip > 0) { skip--; continue; }
          uint32_t age_ms = (newest_ts >= e.timestamp_ms)
                            ? (newest_ts - e.timestamp_ms) : 0;
          kh_render_row(s_list_cont, &e, row, age_ms);
          row++;
      }
  }

  /* ── ZMK_DISPLAY_WIDGET_LISTENER: position events ───────── */

  struct kh_position_state {
      uint32_t position;
      bool     pressed;
      uint8_t  layer;
      uint32_t timestamp_ms;
  };

  static struct kh_position_state kh_position_get_state(const zmk_event_t *eh) {
      const struct zmk_position_state_changed *ev = as_zmk_position_state_changed(eh);
      return (struct kh_position_state){
          .position     = ev->position,
          .pressed      = ev->state,
          .layer        = zmk_keymap_highest_layer_active(),
          .timestamp_ms = (uint32_t)k_uptime_get(),
      };
  }

  static void kh_position_update_cb(struct kh_position_state state) {
      if (!kh_is_recording()) return;
      kh_entry_t e = {
          .type         = state.pressed ? KH_KEY_PRESS : KH_KEY_RELEASE,
          .layer        = state.layer,
          .position     = state.position,
          .keycode      = 0,
          .timestamp_ms = state.timestamp_ms,
      };
      kh_push(&e);
      if (kh_is_active()) kh_screen_rebuild();
  }

  ZMK_DISPLAY_WIDGET_LISTENER(kh_position, struct kh_position_state,
                               kh_position_update_cb, kh_position_get_state)
  ZMK_SUBSCRIPTION(kh_position, zmk_position_state_changed);

  /* ── ZMK_DISPLAY_WIDGET_LISTENER: layer events ───────────── */

  struct kh_layer_state {
      uint8_t  new_layer;
      bool     layer_active;
      uint8_t  highest_layer;
      uint32_t timestamp_ms;
  };

  static struct kh_layer_state kh_layer_get_state(const zmk_event_t *eh) {
      const struct zmk_layer_state_changed *ev = as_zmk_layer_state_changed(eh);
      return (struct kh_layer_state){
          .new_layer     = ev->layer,
          .layer_active  = ev->state,
          .highest_layer = zmk_keymap_highest_layer_active(),
          .timestamp_ms  = (uint32_t)k_uptime_get(),
      };
  }

  static void kh_layer_update_cb(struct kh_layer_state state) {
      if (!kh_is_recording()) return;
      kh_entry_t e = {
          .type         = KH_LAYER_CHANGE,
          .layer        = state.highest_layer,
          .new_layer    = state.new_layer,
          .layer_active = state.layer_active,
          .timestamp_ms = state.timestamp_ms,
      };
      kh_push(&e);
      if (kh_is_active()) kh_screen_rebuild();
  }

  ZMK_DISPLAY_WIDGET_LISTENER(kh_layer, struct kh_layer_state,
                               kh_layer_update_cb, kh_layer_get_state)
  ZMK_SUBSCRIPTION(kh_layer, zmk_layer_state_changed);

  /* ── ZMK_DISPLAY_WIDGET_LISTENER: keycode events ────────── */

  struct kh_keycode_state {
      uint32_t keycode;
      uint8_t  mods;
      bool     pressed;
      uint8_t  layer;
      uint32_t timestamp_ms;
  };

  static struct kh_keycode_state kh_keycode_get_state(const zmk_event_t *eh) {
      const struct zmk_keycode_state_changed *ev = as_zmk_keycode_state_changed(eh);
      return (struct kh_keycode_state){
          .keycode      = ev->keycode,
          .mods         = ev->explicit_modifiers,
          .pressed      = ev->state,
          .layer        = zmk_keymap_highest_layer_active(),
          .timestamp_ms = (uint32_t)k_uptime_get(),
      };
  }

  static void kh_keycode_update_cb(struct kh_keycode_state state) {
      if (!kh_is_recording() || !state.pressed) return;

      /* Walk backward for unresolved KH_KEY_PRESS (within last 4 entries) */
      for (uint8_t i = 0; i < kh_cnt && i < 4; i++) {
          uint8_t idx = (kh_head - 1 - i + KH_RING_BUFFER_SIZE * 2) % KH_RING_BUFFER_SIZE;
          if (kh_buf[idx].type == KH_KEY_PRESS && kh_buf[idx].keycode == 0) {
              kh_buf[idx].keycode = state.keycode;
              kh_buf[idx].mods    = state.mods;
              if (kh_is_active()) kh_screen_rebuild();
              return;
          }
      }

      /* No unresolved press — store standalone */
      kh_entry_t e = {
          .type         = KH_KEY_PRESS,
          .layer        = state.layer,
          .position     = 0xFFFFFFFF,
          .keycode      = state.keycode,
          .mods         = state.mods,
          .timestamp_ms = state.timestamp_ms,
      };
      kh_push(&e);
      if (kh_is_active()) kh_screen_rebuild();
  }

  ZMK_DISPLAY_WIDGET_LISTENER(kh_keycode, struct kh_keycode_state,
                               kh_keycode_update_cb, kh_keycode_get_state)
  ZMK_SUBSCRIPTION(kh_keycode, zmk_keycode_state_changed);

  /* ── Widget init ─────────────────────────────────────────── */

  void kh_widget_init(void) {
      kh_position_init();
      kh_layer_init();
      kh_keycode_init();
  }
  ```

- [ ] **Step 3: Build with `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y`**

  Expected: compiles clean. If `zmk_keymap_highest_layer_active` is not found, check `app/include/zmk/keymap.h` in your ZMK west workspace — the correct name may differ. If so, substitute the correct call throughout.

- [ ] **Step 4: Commit to fork**

  ```bash
  cd /Users/jlee/projects/personal/zmk-dongle-screen
  git add boards/shields/dongle_screen/include/key_history.h \
          boards/shields/dongle_screen/src/widgets/key_history_widget.c
  git commit -m "feat: add ring buffer, display-thread listeners, and LVGL screen for key history"
  git push
  ```

---

## Task 4: ZMK behavior drivers

**Files:**
- Modify: `boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c` (replace stub)

- [ ] **Step 1: Write `boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c`**

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
      uint8_t count      = kh_count();
      uint8_t max_offset = (count > KH_VISIBLE_ROWS) ? (count - KH_VISIBLE_ROWS) : 0;
      uint8_t cur        = kh_get_scroll_offset();
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
  #define KH_TOGGLE_INST(n) \
      DEVICE_DT_INST_DEFINE(n, NULL, NULL, NULL, NULL, \
          APPLICATION, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT, \
          &kh_toggle_driver_api);
  DT_INST_FOREACH_STATUS_OKAY(KH_TOGGLE_INST)
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

- [ ] **Step 2: Build with `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y`**

  Expected: compiles clean. The behavior drivers won't instantiate yet (no DTS nodes exist in the keymap), but the code must compile. If `zmk_display_work_q()` is not found, verify `CONFIG_ZMK_DISPLAY_WORK_QUEUE_DEDICATED=y` is set — it is set as default in `Kconfig.defconfig`. If still missing, check `app/include/zmk/display.h` in your ZMK west workspace.

- [ ] **Step 3: Commit to fork**

  ```bash
  cd /Users/jlee/projects/personal/zmk-dongle-screen
  git add boards/shields/dongle_screen/src/behaviors/key_history_behaviors.c
  git commit -m "feat: add toggle and scroll ZMK behavior drivers for key history"
  git push
  ```

---

## Task 5: Wire up custom_status_screen.c

**Files:**
- Modify: `boards/shields/dongle_screen/src/custom_status_screen.c`

- [ ] **Step 1: Add include and init block**

  Add the include near the top of the file with the other conditional includes:

  ```c
  #if CONFIG_DONGLE_SCREEN_KEY_HISTORY
  #include "key_history.h"
  #endif
  ```

  Then add the init block inside `zmk_display_status_screen()`, after `lv_obj_add_style(screen, &global_style, LV_PART_MAIN);` and before `return screen;`:

  ```c
  #if CONFIG_DONGLE_SCREEN_KEY_HISTORY
      lv_obj_t *hist = kh_screen_create();
      kh_set_screens(screen, hist);
      kh_widget_init();
  #endif
  ```

  The key history block goes immediately after `lv_obj_add_style(...)` and before the existing `#if CONFIG_DONGLE_SCREEN_OUTPUT_ACTIVE` block. Do not modify any existing widget init calls — only add the `#if CONFIG_DONGLE_SCREEN_KEY_HISTORY` block shown above.

- [ ] **Step 2: Build with `CONFIG_DONGLE_SCREEN_KEY_HISTORY=y`**

  Expected: compiles and links clean. The history screen is created at startup. No behavior change yet — the behavior DTS nodes are instantiated only when the keymap includes them (Task 6).

- [ ] **Step 3: Commit to fork and push**

  ```bash
  cd /Users/jlee/projects/personal/zmk-dongle-screen
  git add boards/shields/dongle_screen/src/custom_status_screen.c
  git commit -m "feat: init key history screen at startup when DONGLE_SCREEN_KEY_HISTORY=y"
  git push
  ```

---

## Task 6: Update toucan-zmk config

**Files:**
- Modify: `config/toucan.conf`

- [ ] **Step 1: Replace `config/toucan.conf`**

  The existing file has prospector-specific options that no longer apply. Replace the entire file with:

  ```
  CONFIG_DONGLE_SCREEN_KEY_HISTORY=y
  ```

  The dongle screen's brightness, orientation, and other options have sensible defaults in `Kconfig.defconfig` — no overrides needed unless you want to change them (e.g., `CONFIG_DONGLE_SCREEN_MAX_BRIGHTNESS=80`).

- [ ] **Step 2: Verify the keymap already has behavior DTS nodes**

  Open `config/toucan.keymap` and confirm the `behaviors { }` block already contains `kh_toggle`, `kh_scroll_up`, and `kh_scroll_down` nodes with the correct `compatible` values. They should look like:

  ```dts
  kh_toggle: kh_toggle {
      compatible = "zmk,key-history-toggle";
      label = "KH_TOGGLE";
      #binding-cells = <0>;
  };
  kh_scroll_up: kh_scroll_up {
      compatible = "zmk,key-history-scroll";
      label = "KH_SCROLL_UP";
      #binding-cells = <0>;
      direction = <0>;
  };
  kh_scroll_down: kh_scroll_down {
      compatible = "zmk,key-history-scroll";
      label = "KH_SCROLL_DOWN";
      #binding-cells = <0>;
      direction = <1>;
  };
  ```

  If the `label` property causes a warning in ZMK v0.3, it is safe to remove it.

- [ ] **Step 3: Build the complete dongle firmware**

  Run the local build command (without the `-DCONFIG_DONGLE_SCREEN_KEY_HISTORY=y` override — it now comes from `toucan.conf`). Expected: dongle firmware builds clean with all three behavior DTS nodes instantiated.

- [ ] **Step 4: Commit**

  ```bash
  cd /Users/jlee/projects/personal/toucan-zmk
  git add config/toucan.conf
  git commit -m "feat: enable key history on zmk-dongle-screen, remove prospector config"
  ```

---

## Task 7: End-to-end verification

- [ ] **Step 1: Flash the dongle firmware**

  Use the UF2 from CI or `west flash` locally to flash the dongle.

- [ ] **Step 2: Verify normal screen still works**

  Power cycle the dongle. Confirm the existing `zmk-dongle-screen` widgets (layer, battery, WPM, etc.) appear correctly.

- [ ] **Step 3: Verify history screen opens**

  Type a few keys. Press the FNC layer key + the `&kh_toggle` position. Expected: display switches to the dark "KEY HISTORY" screen with header and list area.

- [ ] **Step 4: Verify ring buffer contents**

  The rows should show the physical key positions you pressed and the resolved keycodes. Layer changes (if any) should appear as green rows with `◆` symbol.

- [ ] **Step 5: Verify scroll**

  Type enough keys to fill more than 10 rows. Open history. Press `&kh_scroll_up` a few times — older entries should appear. Press `&kh_scroll_down` to scroll back toward newest.

- [ ] **Step 6: Verify recording pause**

  While the history screen is open, press several keys including `&kh_scroll_up`. Close history with `&kh_toggle`. Re-open immediately. Expected: the scroll/key presses made while history was open do NOT appear in the list.

- [ ] **Step 7: Verify age fading**

  Type a few keys. Wait 5–10 seconds. Open history. Expected: older entries are visibly dimmer than recent ones.

- [ ] **Step 8: Final push**

  ```bash
  cd /Users/jlee/projects/personal/zmk-dongle-screen
  git push

  cd /Users/jlee/projects/personal/toucan-zmk
  git push
  ```

  > **Reminder:** Confirm before pushing — do not push automatically.

---

## Notes

**`zmk_keymap_highest_layer_active()`** — Used in `get_state` functions. Confirmed present in the `zmk-dongle-screen` codebase's `layer_status.c`. If the linker complains, check `app/include/zmk/keymap.h` in your west workspace.

**`zmk_display_work_q()`** — Declared in `app/include/zmk/display.h`. Available because `Kconfig.defconfig` sets `ZMK_DISPLAY_WORK_QUEUE_DEDICATED` as default. The work items in `key_history_behaviors.c` submit to this queue so `lv_scr_load()` runs on the display thread.

**`ZMK_DISPLAY_WIDGET_LISTENER` internals** — The macro generates a `kh_position_init()` / `kh_layer_init()` / `kh_keycode_init()` function each. Calling `kh_widget_init()` from `custom_status_screen.c` registers all three listeners. The `get_state` function runs on the ZMK main thread (event thread); the update callback runs on the display thread.

**Ring buffer thread safety** — All ring buffer reads and writes happen in the three display-thread callbacks and in `kh_screen_rebuild()` (also display thread). The behavior work items (`do_toggle`, `do_scroll_up`, `do_scroll_down`) also run on the display thread. `kh_is_recording()` uses an atomic because `kh_set_recording()` is called from behavior drivers on the ZMK main thread.

**UTF-8 characters** — `◆` is `\xe2\x97\x86` and `→` is `\xe2\x86\x92`. `CONFIG_LV_TXT_ENC_UTF8=y` is the default in LVGL v8.
