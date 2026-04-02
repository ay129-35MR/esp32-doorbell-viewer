# ESP32-S3 Doorbell Viewer

[![ESPHome](https://img.shields.io/badge/ESPHome-2025-blue?logo=esphome)](https://esphome.io/)  
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Integration-41BDF5?logo=home-assistant)](https://www.home-assistant.io/)  
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)  

[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-ffdd00?logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/ay129)
[![Donate](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://www.paypal.com/donate?business=nyashachipanga%40yahoo.com&currency_code=GBP)

### Using an ESP32-S3 SuperMini, an AliExpress display + encoder module, Home Assistant, PSRAM, and a Linux image server
![Clock Mode](https://github.com/ay129-35MR/esp32-doorbell-viewer/blob/main/IMG_7200.jpeg)
![Camera Mode](https://github.com/ay129-35MR/esp32-doorbell-viewer/blob/main/IMG_7196.jpeg)
![build](https://github.com/ay129-35MR/esp32-doorbell-viewer/blob/main/IMG_7188.jpeg)
![Build](https://github.com/ay129-35MR/esp32-doorbell-viewer/blob/main/IMG_7189.jpeg)
![Family](https://github.com/ay129-35MR/esp32-doorbell-viewer/blob/main/IMG_7190.jpeg)

### TL;DR

This project is a small desktop doorbell camera viewer built from:

- an **ESP32-S3 SuperMini**
- a separate **display + rotary encoder assembly** from AliExpress
- a **custom perfboard motherboard**
- **ESPHome**
- **Home Assistant**
- a **Linux server** that prepares camera snapshots for the ESP

The ESP32-S3 SuperMini is the brains, but **it does not have a screen**. The screen and encoder are a separate hardware module. I mounted both parts onto a simple
perfboard carrier with female 2.54mm headers so the display/encoder module plugs into one side and the SuperMini plugs into the reverse.

The result is a compact little device that:

- sits as a clock/status display when idle
- reacts to Home Assistant motion/person detection
- jumps into a camera page when activity is detected outside
- fetches front/back snapshots from a Linux server
- lets me switch cameras with a rotary encoder
- uses PSRAM for breathing room and for remote screenshot/debugging tooling

### Hardware

### Main controller
- **ESP32-S3 SuperMini**

The SuperMini handles:

- Wi-Fi
- ESPHome runtime
- Home Assistant integration
- display driving
- encoder/button input
- backlight PWM
- OTA updates
- lightweight local web/debug endpoints

### Front panel hardware
- **AliExpress display + rotary encoder module**
- Link: https://www.aliexpress.com/item/1005009880904568.html

This gives the project its front-facing hardware:

- color TFT display
- rotary encoder
- button input hardware
- a compact physical form factor that works nicely for a wall/control-node style device

In software, the display is driven as an **ST7789V 240x320 SPI panel**.

### Custom motherboard
I didn’t start with a PCB. I built a simple motherboard on **perfboard**.

The layout is straightforward:

- female **2.54mm headers** on one side for the display/encoder assembly
- female **2.54mm headers** on the reverse for the ESP32-S3 SuperMini
- point-to-point wiring between them on the perfboard

That made it easy to prototype and swap parts without locking myself into a PCB layout too early.

### Mechanical compromise
The downside is that the cable routing and soldering are a bit ugly.

Right now, the USB cable exits in a way I’m not especially proud of. It sticks out the top because of how I routed everything during assembly. It works, but it’s
visibly a prototype.

This is easy enough to improve:

- use a **90-degree USB cable**
- or fix it properly in **v2** with a custom PCB / revised layout

So the current build is functional and solid electrically, but very much “prototype hardware that escaped into production.”

### What the device does

The node has two main modes:

### 1. Clock / idle page
When idle, the device shows:

- current time
- current date
- night/day brightness behavior
- a visual indication when someone is outside

This makes it useful all the time, not just when motion is detected.

### 2. Camera page
When the user interacts with it, or when Home Assistant reports outside activity, it switches to a full-screen camera page.

That page:

- loads a fresh image for the selected camera
- shows a front/back selector overlay
- shows a timestamp
- allows manual refresh
- times out back to the clock page after inactivity

### Home Assistant’s role

Home Assistant is an important part of the design. It is not just “nice to have.”

The ESP node uses Home Assistant for:

- **time sync**
- **presence/motion/person detection state**
- event-driven behavior

In my setup, the relevant signal is a Home Assistant binary sensor for the front camera area.

When that sensor fires:

- the device switches to the camera page
- the front camera becomes selected
- the backlight goes bright
- a fresh image is fetched

On the idle clock page, the device also indicates when someone is outside.

So Home Assistant is effectively acting as the event/router layer that turns camera analytics or motion detection into something meaningful for the physical device.

That is an important design choice: I’m not trying to make the ESP32-S3 do identity, object detection, or camera analytics locally. Home Assistant is the orchestrator.

### Why this is not a “viewer” running on the screen module

One thing worth being precise about: the project is not based on a self-contained “screen board” that already does everything.

The actual architecture is:

- **display + encoder module** for the front-end hardware
- **ESP32-S3 SuperMini** as the controller
- **perfboard motherboard** as the interconnect
- **Linux server** for image preparation
- **Home Assistant** for event/state logic

The ESP32-S3 is the compute node here. The display assembly is just the human interface hardware.

### Why PSRAM matters

PSRAM is one of the reasons this works comfortably.

In the ESPHome config, I explicitly enable it:

```yaml
psram:
  mode: quad
  speed: 40MHz ...
```

That helps in a few places:

- general memory headroom for a display-heavy ESPHome node
- image handling
- remote screenshot/debugging support
- avoiding the feeling that every feature is one allocation away from crashing the device

### Screenshot/debugging component

I use a custom ESPHome external component that adds an HTTP screenshot endpoint to the device.

That component captures the current display framebuffer and serves it as a BMP over HTTP. It’s extremely useful for:

- remote UI iteration
- documentation
- debugging deployed hardware
- checking exactly what the screen is rendering without physically walking over to it

That workflow is much more pleasant when the S3 has PSRAM available.

So PSRAM here is not marketing fluff. It is directly useful.

### Why the Linux server exists

A big design decision in this project was: do not make the ESP handle the camera pipeline.

Instead of asking the ESP32-S3 to:

- consume camera streams directly
- decode heavy media formats
- resize images
- crop them to fit the display
- convert them into something display-friendly

I offload that to a Linux server on the network.

The Linux server exposes stable endpoints like:

http://<linux-server>:5051/image/front_fluent?width=240&height=320&format=png&fit=cover
http://<linux-server>:5051/image/garden_fluent?width=240&height=320&format=png&fit=cover

That means the server does the expensive work:

- reach the upstream camera source
- prepare the image
- resize to the target screen
- crop for portrait fit
- convert to PNG
- expose a simple URL for the ESP

Then the ESP just downloads exactly the image it needs.

This is a much more practical split of responsibilities.

### Why this architecture works well

This split has a few major advantages.

### Linux server does the heavy lifting

The Linux box is much better suited for:

- camera ingestion
- image transforms
- future caching
- API normalization
- integrating with upstream camera systems

### Home Assistant handles automation and context

Home Assistant is where I already have:

- motion/person events
- schedules
- time
- presence logic
- automation flows

So it makes sense to let HA drive the “when should the node wake/show camera” behavior.

### ESP32-S3 handles local interaction

The ESP is then left to do the embedded-friendly part:

- show a good UI
- react quickly to local controls
- fetch pre-rendered images
- expose basic web/debug endpoints
- stay reliable

That is a much better fit than trying to make the microcontroller become a full media endpoint.

### User interaction

The physical interaction is intentionally simple.

### Navigation button

Used to move between the idle page and the camera page.

### Rotary encoder

Used to switch between:

- front camera
- back/garden camera

### Encoder push

Used to refresh the selected camera image.

That gives the device a clean appliance-like behavior instead of turning it into a menu-heavy gadget.

### Visual behavior

### Idle mode

- black background
- large clock
- date
- dimmer night mode
- outside/person indication from Home Assistant

### Camera mode

- full-screen portrait snapshot
- selector overlay for front/back
- timestamp overlay
- loading state while fetching

It is intentionally minimal and readable.

### The prototype hardware story

This is not a polished enclosure-first build.

It started as:

- a SuperMini
- a display/encoder module
- perfboard
- headers
- soldered routing

That was deliberate.

I wanted to validate:

- the UI
- the HA flow
- the camera snapshot flow
- the ergonomics of the encoder
- whether the concept was actually useful day to day

before investing time in a proper PCB.

The answer is yes, it’s useful, which means the ugly parts are now worth cleaning up in v2.

### Version 2 ideas

The obvious next steps are:

- design a proper PCB instead of perfboard
- fix USB cable orientation properly
- improve enclosure/cable exit
- tighten the mounting arrangement
- maybe expose cleaner expansion points
- refine the front-panel mechanical fit

But the current prototype already proves the architecture.

### Why this project has held up

A lot of small ESP display projects are fun for a week and then stop being useful.

This one has held up better because it does a specific job well:

- always-visible clock/status node
- event-driven camera jump on outside motion/person detection
- local physical control
- remote screenshot/debugging
- sensible separation between ESP, HA, and Linux server responsibilities

