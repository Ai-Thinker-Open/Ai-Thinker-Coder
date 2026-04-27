# lvgl - Light and Versatile Graphics Library

## Overview

lvgl (Light and Versatile Graphics Library) is a free and open-source graphics library for embedded systems. It provides everything needed to create embedded GUIs with easy-to-use graphical elements, beautiful visual effects, and a low memory footprint.

## Location

```
/home/seahi/workspase/Ai-Thinker-WB2/components/stage/lvgl/
```

## Key Files

| File | Description |
|------|-------------|
| `lvgl.h` | Main header, includes all components |
| `lv_conf.h` | Configuration file |
| `LICENCE.txt` | License (SLA) |
| `lv_srcs/` | LVGL source code |
| `lv_device/` | Device drivers (display, input) |

### lv_srcs Structure

| Directory | Description |
|-----------|-------------|
| `core/` | Core functionality (objects, display, input) |
| `widgets/` | UI widgets (button, label, slider, etc.) |
| `font/` | Font handling |
| `hal/` | Hardware abstraction layer |
| `misc/` | Miscellaneous (memory, timers, math) |
| `draw/` | Drawing primitives |
| `extra/` | Extra widgets |
| `lv_api_map.h` | API version mapping |

### Device Drivers (lv_device/)

| Directory | Description |
|-----------|-------------|
| `lv_port_disp.h` | Display port header |
| `lv_port_indev.h` | Input device port header |
| `lv_port_fs.h` | Filesystem port header |
| `st7789/` | ST7789 display driver |
| `st7796s/` | ST7796S display driver |
| `ssd1306/` | SSD1306 OLED display driver |
| `cst816/` | CST816 touch controller driver |

## Version

```
LVGL_VERSION_MAJOR 8
LVGL_VERSION_MINOR 3
LVGL_VERSION_PATCH 4
```

Version check macro:

```c
#define LV_VERSION_CHECK(x,y,z) (x == LVGL_VERSION_MAJOR && \
    (y < LVGL_VERSION_MINOR || (y == LVGL_VERSION_MINOR && z <= LVGL_VERSION_PATCH)))
```

## Configuration

Key settings in `lv_conf.h`:

### Display Driver Selection

```c
#define LV_DISPLAY_SSD1306
// #define LV_DISPLAY_ST7796S
// #define LV_DISPLAY_ST7789
```

### Input Driver Selection

```c
#define LV_INDEV_CST816
```

### Display Resolution

```c
#define MY_DISP_HOR_RES    128  // Horizontal resolution
#define MY_DISP_VER_RES    64   // Vertical resolution
```

### Color Depth

```c
// For SSD1306 (monochrome):
#define LV_COLOR_DEPTH 1

// For ST7796S/ST7789 (color):
#define LV_COLOR_DEPTH 16
```

### Memory Settings

```c
#define LV_MEM_CUSTOM 0
#define LV_MEM_SIZE (48U * 1024U)  // 48KB for lv_mem_alloc()
```

---

## Core Concepts

### Object Hierarchy

All UI elements are objects (`lv_obj_t`) arranged in a tree hierarchy:

```
Screen (lv_obj_t *)
  └── Container (lv_obj_t *)
       ├── Button (lv_btn_t)
       ├── Label (lv_label_t)
       └── Slider (lv_slider_t)
```

### Object Types

| Widget | Header | Description |
|--------|--------|-------------|
| Arc | `lv_arc.h` | Circular arc widget |
| Bar | `lv_bar.h` | Progress bar |
| Button | `lv_btn.h` | Clickable button |
| Button matrix | `lv_btnmatrix.h` | Grid of buttons |
| Calendar | (extra) | Calendar widget |
| Canvas | `lv_canvas.h` | Drawing canvas |
| Checkbox | `lv_checkbox.h` | Checkbox control |
| Color wheel | (extra) | Color picker |
| Dropdown | `lv_dropdown.h` | Drop-down list |
| Image | `lv_img.h` | Image display |
| Keyboard | (extra) | On-screen keyboard |
| Label | `lv_label.h` | Text label |
| LED | (extra) | LED indicator |
| Line | `lv_line.h` | Line drawing |
| List | (extra) | Scrollable list |
| Menu | (extra) | Menu widget |
| Meter | (extra) | Gauge/meter |
| Message box | (extra) | Modal dialog |
| Roller | `lv_roller.h` | Rolling selector |
| Slider | `lv_slider.h` | Slider control |
| Spinbox | (extra) | Number input |
| Spinner | (extra) | Loading animation |
| Switch | `lv_switch.h` | Toggle switch |
| Table | `lv_table.h` | Table widget |
| Tabview | (extra) | Tab container |
| Textarea | `lv_textarea.h` | Multi-line text input |
| Tileview | (extra) | Tile-based view |
| Window | (extra) | Window with header |

---

## Core API

### Display Initialization

```c
#include "lvgl.h"
#include "lv_device/lv_port_disp.h"

void my_disp_init(void) {
    lv_port_disp_init();
}
```

### Screen and Objects

```c
// Create a screen
lv_obj_t *scr = lv_obj_create(NULL);

// Create an object on the screen
lv_obj_t *btn = lv_btn_create(scr);

// Delete an object
lv_obj_del(btn);

// Get parent/child
lv_obj_t *parent = lv_obj_get_parent(child);
lv_obj_t *child = lv_obj_get_child(parent, idx);
```

### Widget Creation

```c
// Button
lv_obj_t *btn = lv_btn_create(parent);
lv_obj_set_size(btn, 100, 50);
lv_obj_align(btn, LV_ALIGN_CENTER, 0, 0);

// Label
lv_obj_t *label = lv_label_create(parent);
lv_label_set_text(label, "Hello");
lv_obj_align(label, LV_ALIGN_CENTER, 0, 0);

// Slider
lv_obj_t *slider = lv_slider_create(parent);
lv_slider_set_range(slider, 0, 100);
lv_slider_set_value(slider, 50, LV_ANIM_ON);

// Switch
lv_obj_t *sw = lv_switch_create(parent);
lv_obj_align(sw, LV_ALIGN_CENTER, 0, 50);

// Checkbox
lv_obj_t *cb = lv_checkbox_create(parent);
lv_checkbox_set_text(cb, "Option");

// Dropdown
lv_obj_t *dd = lv_dropdown_create(parent);
lv_dropdown_set_options(dd, "Option 1\nOption 2\nOption 3");

// Roller
lv_obj_t *roller = lv_roller_create(parent);
lv_roller_set_options(roller, "Option 1\nOption 2\nOption 3", LV_ROLLER_MODE_NORMAL);
```

### Styling

```c
// Style definition
static lv_style_t style_btn;
lv_style_init(&style_btn);
lv_style_set_bg_color(&style_btn, lv_palette_main(LV_PALETTE_BLUE));
lv_style_set_radius(&style_btn, 10);

// Apply style
lv_obj_add_style(btn, &style_btn, 0);

// Style properties
lv_style_set_bg_color(style, color);
lv_style_set_bg_opa(style, opa);
lv_style_set_text_color(style, color);
lv_style_set_pad_all(style, 10);
lv_style_set_radius(style, radius);
```

### Layout

```c
// Alignment
lv_obj_align(obj, align, x_ofs, y_ofs);

// Align to another object
lv_obj_align_to(obj, obj_to_align_to, align, x, y);

// Flex layout
lv_obj_set_flex_flow(parent, LV_FLEX_FLOW_ROW);
lv_obj_set_flex_align(parent, LV_FLEX_ALIGN_START, LV_FLEX_ALIGN_CENTER, LV_FLEX_ALIGN_CENTER);

// Grid layout
lv_obj_set_grid_dsc_array(parent, col_dsc, row_dsc);
lv_obj_set_grid_cell(obj, LV_GRID_ALIGN_START, col, col_span, 
                              LV_GRID_ALIGN_START, row, row_span);
```

### Events

```c
static void btn_event_cb(lv_event_t *e) {
    lv_obj_t *target = lv_event_get_target(e);
    lv_event_code_t code = lv_event_get_code(e);
    
    if (code == LV_EVENT_CLICKED) {
        // Button clicked
    }
}

lv_obj_add_event_cb(btn, btn_event_cb, LV_EVENT_ALL, NULL);
```

### Timer

```c
static void timer_callback(lv_timer_t *timer) {
    // Called periodically
}

lv_timer_t *timer = lv_timer_create(timer_callback, 1000, NULL);  // 1000ms period
lv_timer_del(timer);
```

### Memory

```c
#include "lv_srcs/misc/lv_mem.h"

void *ptr = lv_mem_alloc(64);
lv_mem_free(ptr);
```

### Input Devices

```c
#include "lv_device/lv_port_indev.h"

lv_indev_t *indev = lv_indev_get_next(NULL);
// Configure touch in lv_port_indev.c
```

---

## Display Drivers

### SSD1306 (OLED)

```c
// Pin configuration (lv_conf.h)
#define OLED_IIC_SCL 12
#define OLED_IIC_SDA 3
#define LV_DISPLAY_ORIENTATION_LANDSCAPE 
// or
// #define LV_DISPLAY_ORIENTATION_LANDSCAPE_INVERTED 

// Resolution
#define MY_DISP_HOR_RES    128
#define MY_DISP_VER_RES    64
```

### ST7789 (TFT)

```c
// Pin configuration
#define ST7789_DC 4
#define ST7789_CS 5
#define ST7789_RST 14
#define ST7789_CLK 3
#define ST7789_MOSI 12
```

### ST7796S (TFT)

```c
// SPI pins
#define ST7796_SPI_SS 5
#define ST7796_SPI_RST 4
#define ST7796_SPI_DC 11
#define ST7796_SPI_MOSI 12
#define ST7799_SPI_CLK 3
#define ST7796_SPI_BL 14

// Resolution
#define MY_DISP_HOR_RES    480
#define MY_DISP_VER_RES    320
```

---

## Input Drivers

### CST816 (Touch)

```c
// Enable in lv_conf.h
#define LV_INDEV_CST816

// Configuration in cst816.h
```

---

## Usage Example

```c
#include "lvgl.h"
#include "lv_device/lv_port_disp.h"
#include "lv_device/lv_port_indev.h"

static void btn_event_cb(lv_event_t *e) {
    lv_obj_t *btn = lv_event_get_target(e);
    
    if (lv_event_get_code(e) == LV_EVENT_CLICKED) {
        static uint32_t count = 0;
        lv_obj_t *label = lv_obj_get_child(btn, 0);
        if (label) {
            char buf[32];
            snprintf(buf, sizeof(buf), "%"LV_PRIu32, ++count);
            lv_label_set_text(label, buf);
        }
    }
}

void ui_init(void) {
    // Initialize display and input
    lv_port_disp_init();
    lv_port_indev_init();
    
    // Create screen
    lv_obj_t *scr = lv_obj_create(NULL);
    lv_scr_load(scr);
    
    // Create button with label
    lv_obj_t *btn = lv_btn_create(scr);
    lv_obj_set_size(btn, 120, 50);
    lv_obj_align(btn, LV_ALIGN_CENTER, 0, 0);
    lv_obj_add_event_cb(btn, btn_event_cb, LV_EVENT_ALL, NULL);
    
    lv_obj_t *label = lv_label_create(btn);
    lv_label_set_text(label, "Click: 0");
    lv_obj_center(label);
}
```

---

## Dependencies

The component uses FreeRTOS for timer functionality. Ensure `configUSE_TICKLESS_IDLE` and `INCLUDE_vTaskDelay` are properly configured in your FreeRTOS settings.

---

## Porting Notes

### Display Port

Implement the following functions in your `lv_port_disp.c`:

```c
void lv_port_disp_init(void);
void my_disp_flush(lv_disp_t *disp, const lv_area_t *area, uint8_t *color_p);
```

### Input Port

Implement for touch input:

```c
void lv_port_indev_init(void);
bool my_touch_read(lv_indev_t *indev, lv_indev_data_t *data);
```
