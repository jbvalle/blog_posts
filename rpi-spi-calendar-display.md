# LVGL Calendar Display

*Technical walkthrough of an embedded Linux agenda display using LVGL, direct SPI/GPIO control, and Google Calendar ICS synchronisation.*

---

## Overview

This project is a dedicated 320 × 240 agenda display for embedded Linux. It retrieves a Google Calendar ICS feed, evaluates the relevant events, and renders the current agenda on an ILI9341 TFT display.

The focus was not only on making a graphical interface, but on integrating the complete embedded system: Linux device interfaces, a custom display driver, LVGL rendering, network communication, calendar parsing, recurrence handling, and service deployment.

The reference platform is a Raspberry Pi running 64-bit Raspberry Pi OS. The application itself is written so that its SPI device, GPIO wiring, display orientation, backlight, and calendar URL can be configured at runtime.

---

## System Architecture

```text
Google Calendar ICS feed
          │
          ▼
libcurl worker thread ──► ICS parser + recurrence expansion
          │                         │
          │                   agenda snapshot
          ▼                         │
     connection state ◄─────────────┘
          │
          ▼
LVGL UI thread ──► partial RGB565 render buffer ──► ILI9341 SPI display
```

The worker thread never modifies LVGL objects. It downloads and parses calendar data, then publishes a protected agenda snapshot. The main thread reads that snapshot, updates the screen state, and runs the LVGL timer handler. Keeping rendering and network activity separate prevents a slow request from blocking the interface.

---

## Software Stack

| Layer | Technology | Role |
|------|------------|------|
| Operating system | Raspberry Pi OS 64-bit / Embedded Linux | Provides SPI, GPIO, networking, and systemd services |
| Application language | C11 | Main application, parser, UI, and hardware abstraction |
| UI framework | LVGL 9.5 | Object-based UI layout and partial rendering |
| Display transport | Linux spidev + GPIO character-device v2 API | ILI9341 command/data transfers, reset, and backlight control |
| Network client | libcurl | ICS download with timeout and response-size limit |
| Build system | CMake + FetchContent | Reproducible build with a pinned LVGL release |
| Runtime service | systemd | Starts after network availability and restarts on failure |

The CMake configuration fetches LVGL version 9.5.0 explicitly, links `CURL::libcurl` and `Threads::Threads`, enables C11, and uses compiler warnings. This keeps the build self-contained instead of relying on an unpinned framework version installed on the target.

---

## Display Driver and Linux Hardware Access

The display layer is implemented directly in C for an ILI9341 controller. The driver opens the SPI device, configures SPI mode 0, 8-bit words, and the requested clock speed, then performs the controller initialisation sequence.

| Interface | Default configuration | Purpose |
|-----------|-----------------------|---------|
| SPI | `/dev/spidev0.0`, 32 MHz | Transfers ILI9341 commands and RGB565 pixel data |
| D/C GPIO | BCM 25 | Selects command or data mode |
| Reset GPIO | BCM 24 | Hardware reset of the display controller |
| Backlight GPIO | BCM 18 | Enables the display backlight when connected |
| Rotation | 90° | Uses the ILI9341 MADCTL register for landscape output |

The GPIO implementation uses the modern Linux GPIO character-device v2 API rather than shell commands or a board-specific library. Pixel transfers are sent in bounded 4096-byte SPI chunks, and the driver verifies the requested display area before writing it. The display controller, wiring, rotation, SPI speed, and backlight can all be changed through command-line options.

The optional touch controller is deliberately not opened. This is a display-only product, so avoiding unused touch polling reduces integration scope and keeps the runtime behaviour predictable.

---

## LVGL Implementation

LVGL is initialised after the hardware display is available. The application provides LVGL with the actual width and height reported by the hardware layer, a monotonic millisecond tick source, a single RGB565 draw buffer, and a custom flush callback.

```c
static uint8_t draw_buffer[320 * DRAW_BUFFER_LINES * 2];

lv_display_t *display = lv_display_create(hardware.width, hardware.height);
lv_display_set_buffers(display, draw_buffer, NULL, sizeof(draw_buffer),
                       LV_DISPLAY_RENDER_MODE_PARTIAL);
lv_display_set_color_format(display, LV_COLOR_FORMAT_RGB565);
lv_display_set_flush_cb(display, display_flush);
```

`DRAW_BUFFER_LINES` is set to 40. At 320 pixels wide and two bytes per RGB565 pixel, the application uses a 25,600-byte partial render buffer instead of allocating a full frame buffer. LVGL renders only the changed regions and calls the flush callback for each area.

The flush callback performs the byte swap required by the ILI9341 data format, passes the rectangle to the SPI driver, swaps the buffer back for LVGL, and finally signals that the flush is complete. This is the bridge between a high-level LVGL object tree and the physical display controller.

The LVGL configuration is intentionally small:

```c
#define LV_COLOR_DEPTH 16
#define LV_FONT_MONTSERRAT_14 1
#define LV_FONT_MONTSERRAT_16 1
#define LV_FONT_MONTSERRAT_20 1
```

The UI is built from native LVGL objects: a root screen, heading and date labels, a connection-status indicator, and three reusable agenda-card structures. Each card contains a time box, day label, title, metadata label, and colour accent. Labels use clipped long-text mode so an overlong event title cannot disturb the layout.

---

## Google Calendar Integration and ICS Parsing

The default calendar source is a public Google Calendar ICS URL. The URL can be replaced at runtime with `--calendar-url`, which allows the same firmware to be used with another compatible calendar source without recompiling.

The client refreshes in a dedicated POSIX thread every 60 seconds. libcurl uses an 8-second request timeout and limits the response to 1 MiB before parsing. The parser handles:

- Standard one-off `VEVENT` entries
- Timed and all-day events
- Folded ICS lines and escaped calendar text
- Daily, weekly, monthly, and yearly recurrence rules
- Recurrence exceptions (`EXDATE`) and modified recurring instances (`RECURRENCE-ID`)
- Current events as well as upcoming events within a 45-day horizon

Only the three most relevant events are retained for the display. This keeps memory use bounded and matches the available screen space. Timed events are formatted in the system timezone; the target system can therefore be configured with the appropriate local timezone, such as `Europe/Vienna`.

---

## UI State Handling

The display communicates the state of the data source without making the operator interpret debug messages.

| State | UI behaviour |
|-------|--------------|
| First request in progress | `CONNECTING` status with a loading message |
| Successful refresh | Timestamped sync state and next agenda entries |
| Refresh in progress after a success | Existing agenda stays visible while the status shows `SYNCING` |
| Refresh failure after a success | Last valid agenda remains on screen and status changes to `OFFLINE` |
| No upcoming items | A dedicated empty state for the next 45 days |

An event currently in progress is highlighted and shown as `NOW`. This is a small detail, but it turns the UI from a static schedule into a live status display.

---

## Build and Deployment

The project is built with CMake. On first configuration, CMake downloads the pinned LVGL source; later builds reuse the local dependency.

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
./build/spi_calendar --speed 16000000
```

An example systemd unit is included for unattended operation. It waits for `network-online.target`, runs the application as a user in the `spi` and `gpio` groups, and uses `Restart=on-failure` to recover from an unexpected exit.

```ini
[Unit]
Wants=network-online.target
After=network-online.target

[Service]
SupplementaryGroups=spi gpio
ExecStart=/home/pi/003_calender/build/spi_calendar --speed 16000000
Restart=on-failure
```

The lower initial SPI speed is useful for first hardware bring-up. After confirming display reliability, the standard 32 MHz configuration can be used.

---

## Engineering Scope Demonstrated

This project demonstrates the complete path from a connected data source to a deployed embedded user interface:

- Embedded Linux device access using SPI and GPIO APIs
- Low-level display controller initialisation and RGB565 pixel transfer
- LVGL display-port integration and memory-conscious partial rendering
- C-based UI composition, styling, clipping, and live state updates
- Threaded network communication with safe hand-off to the UI thread
- ICS parsing, recurrence expansion, exception handling, and local-time formatting
- Reproducible CMake builds and systemd-based deployment

The reference configuration uses a public ICS link, which should be treated as a secret URL because anyone who obtains it can read the events it exposes. For a customer deployment, the display architecture can instead be connected to a scoped calendar, private backend, or another approved data source.
