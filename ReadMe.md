# NeoPixelBus

> **This is a fork** of [Makuna/NeoPixelBus](https://github.com/Makuna/NeoPixelBus) (`master` at `882b804`, the
> three commits after the `2.8.4` release, plus one patch for
> [issue #921](https://github.com/Makuna/NeoPixelBus/issues/921), see
> [Patch for Makuna/NeoPixelBus#921](#patch-for-makunanepixelbus921---esp32-rmt-memory-blocks-and-refill-interrupt)).
> Only the two ESP32 RMT files differ from upstream, the library keeps its name so it can replace the upstream
> package as a dependency (`lib_deps = https://github.com/sascha432/NeoPixelBus.git#kfc-rmt1`, branch
> `kfc-rmt-mem-blocks`).

[![Donate](https://img.shields.io/badge/paypal-donate-yellow.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=6AA97KE54UJR4)

Arduino NeoPixel library

A library to control one wire protocol RGB, RGBW, and RGBWW leds like APA106, SK6812, WS2811, WS2812 and WS2813 that are commonly refered to as NeoPixels and two wire protocol RGB like Lpd8806, APA102 and SK9822 commonly refered to as DotStars.
Supports most Arduino platforms.  

For quick questions and support:  
* [Try the new Github Discussions](https://github.com/Makuna/NeoPixelBus/discussions)  
* [Discord NeoPixelBus Invitation](https://discord.gg/c6FrysvZyV) or if you are already a member of [Discord Server NeoPixelBus](https://discord.com/channels/789177382221119519/789177382221119521)  
* Or jump on Gitter   
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/Makuna/NeoPixelBus?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)  

For bugs, make sure there isn't an active issue and then create one.

## Patch for Makuna/NeoPixelBus#921 - ESP32 RMT memory blocks and refill interrupt

Upstream issue: [Makuna/NeoPixelBus#921](https://github.com/Makuna/NeoPixelBus/issues/921) - ESP32 with an
active WiFi/BLE connection produces mid-frame LED latches ("ghost pixels", shifted patterns, a "moving start
LED"). Proposed upstream fixes: PR [#922](https://github.com/Makuna/NeoPixelBus/pull/922) (closed, the same
idea for `master`) and PR [#923](https://github.com/Makuna/NeoPixelBus/pull/923) (open, it targets the `CORE3`
branch, which uses the ESP-IDF v5 RMT driver). This fork patches the **legacy ESP-IDF v3/v4** driver that
`master` and 2.8.x use, because the ESP32 Arduino core 2.x (ESP-IDF 4.4) is what the affected projects build
against. The base is `master` at `882b804` ("Delayed malloc (#911)") - the three commits after the `2.8.4`
release (#894, #910, #911) are included; the `2.8.4` tag itself is unaffected and does not contain the patch.

**Cause.** `NeoEsp32RmtMethodBase::Initialize()` hardcodes `config.mem_block_num = 1` (64 RMT items = ~80 us of
WS2812 data) and installs the ISR with `ESP_INTR_FLAG_LOWMED` on IDF >= 4.3. The driver refills the channel
memory from its threshold interrupt, which is set to **half** of that memory, so the interrupt has to fire
within **~40 us** for the whole frame. TX ping-pong is always enabled, so an interrupt that is late makes the
hardware **transmit stale memory a second time**: the data line gets a gap, the strip latches in the middle of
the frame and the rest of the frame is shifted. WiFi, flash cache-off windows (NVS/LittleFS writes) and DMA
peripherals cause such delays.

**Fix (2 files, both inert unless the application opts in).**

1. `NEOPIXELBUS_RMT_INT_FLAGS` is wrapped in `#ifndef` and defaults to
   `ESP_INTR_FLAG_IRAM | ESP_INTR_FLAG_LEVEL3` - the values FastLED's ESP32 RMT driver uses. Override with
   `-D NEOPIXELBUS_RMT_INT_FLAGS=...` for A/B tests.
2. `mem_block_num` is no longer hardcoded: it reads the new `uint8_t gNeoPixelBusRmtMemBlocks` (defined in
   `NeoEsp32RmtMethod.cpp`, default `1` = upstream behavior). The application sets it before the bus is
   initialized. A channel with N blocks occupies the memory of the N-1 channels that follow it, so the
   channels have to be spaced by N: the ESP32 has 8 blocks in total -> 8 for one strip (~320 us of refill
   headroom), 4 for two strips (~160 us). FastLED uses 2 blocks per channel; its 4.x driver sizes the buffer
   from the network state instead.

**Measured** on an ESP32 (Arduino core 4.20017 / ESP-IDF 4.4.7, WS2812x @ 800 kHz, 320 pixels): with 1 block
the strip flickers, with all 8 blocks it is clean (63.5 fps instead of 48.6 fps). Same conclusion as the issue
reporter: the refill window has to cover the longest interrupt delay of the system - ~320 us was enough here,
~160 us (two channels sharing the memory) was not.

```ini
# use the fork as a PlatformIO dependency (tag kfc-rmt1, branch kfc-rmt-mem-blocks)
lib_deps = https://github.com/sascha432/NeoPixelBus.git#kfc-rmt1
```

License is unchanged, LGPL-3.0 (see `COPYING`). The patch keeps the upstream structure so it can be dropped as
soon as an equivalent fix is released upstream.

## Documentation
[See Wiki](https://github.com/Makuna/NeoPixelBus/wiki)

## Installing This Library (preferred, you just want to use it)
Open the Library Manager and search for "NeoPixelBus by Makuna" and install

## Installing This Library From GitHub (advanced, you want to contribute)
Create a directory in your Arduino\Library folder named "NeoPixelBus"
Clone (Git) this project into that folder.  
It should now show up in the import list when you restart Arduino IDE.





