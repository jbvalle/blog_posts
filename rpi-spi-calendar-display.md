# LVGL Calendar Display

*An embedded Linux calendar interface with live Google Calendar synchronisation — one small display, current information, and no app to open.*

---

## Overview

The best embedded products usually make one task feel effortless. This project turns a Raspberry Pi and a compact 320 × 240 ILI9341 display into a dedicated agenda screen that keeps the next important events visible at a glance.

It is designed as a small, focused Linux user interface rather than a scaled-down desktop application. The screen shows the current date, connection state, and up to three current or upcoming calendar items. There are no menus to navigate and no touch layer to maintain — just the information that matters, available when it is needed.

For a desk, workshop, meeting room, or shared workspace, this kind of device has a simple purpose: reduce the friction between planning something and seeing it.

---

## Project at a Glance

| Area | Implementation |
|------|----------------|
| Platform | Raspberry Pi running 64-bit Raspberry Pi OS |
| Display | 320 × 240 ILI9341 TFT over SPI |
| User interface | LVGL 9.5, rendered in RGB565 |
| Application | C11 with CMake build configuration |
| Calendar source | Public Google Calendar ICS feed, configurable at runtime |
| Refresh strategy | Background update every 60 seconds |
| Operation | Optional systemd service for start-at-boot |

---

## The Goal: Make Information Visible

A calendar is useful only if people actually look at it. Phone notifications are easy to dismiss, and opening a browser or app adds a small interruption every time. The aim here was different: create a quiet display that communicates the next thing to do without demanding attention.

The interface is intentionally small and direct. A clean heading, today’s date, a compact sync status, and three agenda cards give the screen a clear visual hierarchy. Events that are currently in progress receive a distinct accent, while upcoming entries remain easy to scan from a short distance.

This approach is practical for customers and stakeholders because it keeps the experience understandable. The device does not need a manual before it becomes useful.

---

## From Calendar Feed to Display

The reference configuration fetches a public Google Calendar ICS feed using libcurl. Downloading happens in a worker thread, so a slow network request never stops the user interface from updating. Once a new feed has been processed, the latest agenda is passed to the LVGL view for rendering.

| Step | What happens |
|------|--------------|
| 1. Fetch | The public ICS feed is retrieved in the background. |
| 2. Parse | Events are read from the calendar data, including folded lines and escaped text. |
| 3. Expand | One-off and common recurring events are evaluated for the upcoming agenda. |
| 4. Select | The display keeps the most relevant current or upcoming three items within a 45-day horizon. |
| 5. Present | The UI refreshes while preserving a clear connection and sync status. |

The parser accounts for all-day events as well as common daily, weekly, monthly, and yearly recurrence rules. It also respects exceptions and modified instances in recurring series. This is important because calendar data is rarely as simple as a list of one-time appointments.

---

## Reliability Is Part of the Interface

An embedded display should remain useful when the network is not perfect. Before the first successful connection, the screen clearly shows that the calendar is unavailable. After a successful update, the last valid agenda remains visible if a later refresh fails, while the device continues retrying automatically.

The UI exposes three understandable states: connecting, synced, and offline. This provides useful feedback without filling the display with technical detail. A short request timeout and bounded download size keep the network operation controlled, while the display loop remains responsive throughout.

---

## Calendar Data and Privacy

The reference implementation uses a public Google Calendar ICS feed, which keeps the integration lightweight and removes the need to store Google credentials on the device. A public calendar URL should be treated like a secret link: anyone who obtains it can read the events it exposes.

For a customer deployment, the same display layer can use a deliberately scoped calendar, a private integration service, or another approved data source. This makes the product concept flexible while keeping the data-handling decision visible from the start.

---

## Hardware and Display Integration

The ILI9341 display is driven directly through Linux SPI and GPIO interfaces. The application initializes the controller, configures the display orientation, and transfers only the parts of the frame that LVGL needs to redraw. Pixel data is handled in RGB565 format, which fits the display hardware and keeps the system lightweight.

This project intentionally does not open or poll the optional touch controller. Removing input from the scope makes the device more reliable for its intended use: a read-only status display that starts, updates, and recovers without interaction.

The hardware configuration can be adapted at runtime for a different SPI device, GPIO wiring, display rotation, backlight behaviour, or calendar URL. That makes the core application reusable instead of being locked to one desk setup.

---

## Ready for Everyday Use

The project uses CMake with a pinned LVGL release for repeatable builds. An example systemd unit starts the display only after networking is available and restarts it if it exits unexpectedly. This is the difference between a successful prototype and something that can quietly run every day.

The result is a compact embedded Linux product that brings together firmware-level hardware control, a modern graphical interface, networked data, and operational thinking in one focused system.

---

## Where This Can Go Next

The same architecture can be adapted beyond a personal calendar. With a different data source and screen layout, it can become a meeting-room schedule, production status panel, booking display, lab dashboard, or other small information appliance.

That is what makes projects like this interesting: the hardware is compact, but the product idea is flexible. If you are looking for a focused embedded interface that turns live information into something simple and useful, feel free to [get in touch](mailto:johnbryan.valle@j-techworks.net).
