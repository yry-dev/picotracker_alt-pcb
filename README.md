# Picotracker Alt-PCB

~~**Do not fabricate! There could be a major mistake around MIDI!!**~~

**[Alex fixed it!](https://github.com/ijnekenamay/picotracker_alt-pcb/pull/2) We should be able to go ahead with production now.**

This project is about [democloid picoTracker's](https://github.com/democloid/picoTracker/) alternative PCB.
Inexpensive handheld devices can be built to run LittleGPTracker(a.k.a piggytracker).
It basically follows the original DIY version, with a few modifications of my own. They are described below.

[You can see it in motion here.](https://www.instagram.com/p/C2BvngaruZN/?utm_source=ig_web_copy_link&igsh=MzRlODBiNWFlZA==)

<img src="https://raw.githubusercontent.com/ijnekenamay/picotracker_alt-pcb/main/images/picotracker-front.jpg" width="400"><img src="https://raw.githubusercontent.com/ijnekenamay/picotracker_alt-pcb/main/images/picotracker-back.jpg" width="400">

## Feature

- The two PCBs act as an enclosure by sandwiching the components together.
- I wanted a module for the charging function and plenty of memory, so I went with the [Pimoroni Pico LiPo](https://shop.pimoroni.com/products/pimoroni-pico-lipo).
- I wanted a micro-speaker like the DirtywaveM8, so I added one... and a Class D amplifier module.
- A speaker kill switch has been added.
- This also uses Kailh Choc V1 for the keycaps because of my personal taste.
- You can change any specification by yourself if you have access to Kicad.
- [See image for values of components around MIDI.](https://raw.githubusercontent.com/ijnekenamay/picotracker_alt-pcb/main/images/MIDI_resistor_value.jpg)
- I designed a back panel that could be output with a 3D printer because I felt the microspeaker needed box resonance. Also, if you would like to try it, please see the STL data. Feedback is welcome.
- <img src="https://raw.githubusercontent.com/ijnekenamay/picotracker_alt-pcb/main/images/3d_BP.png" width="400">

## BOM

| Value | Qty | Avg unit price¹ | Mouser | DigiKey | Notes |
| --- | --- | --- | --- | --- | --- |
| Pimoroni Pico LiPo 16MB | 1 | ~$8–16 | — | — | [Pimoroni](https://shop.pimoroni.com/products/pimoroni-pico-lipo) (~$15.95, [PiShop US](https://www.pishop.us/product/pimoroni-pico-lipo-16mb/)) · [JP](https://akizukidenshi.com/catalog/g/g116997/). Equivalent: [Waveshare RP2040-Plus 16MB](https://www.amazon.com/Waveshare-RP2040-Plus-High-Performance-Microcontroller-Compatible/dp/B09KZKKWJT) ([Waveshare](https://www.waveshare.com/rp2040-plus.htm)), ~$8 — Pico-pinout, USB-C, onboard LiPo charge, 16MB (use the **no-header/castellated** version). |
| ILI9341 3.2" display | 1 | ~$12–18 | — | — | [AliExpress](https://www.aliexpress.us/item/3256802819098352.html). Equivalents: generic 3.2" ILI9341 SPI 320×240 on [Amazon](https://www.amazon.com/ALMOCN-ILI9341-Display-320x240-Screen/dp/B0C88S6FK7) (ALMOCN/Hosyond/ACEIRMC, ~$15) — **verify the 8-pin SPI header order matches J7**. Tayda has only a [2.8" ILI9341](https://www.taydaelectronics.com/2-8-inch-spi-display-screen-tft-with-touch-ili9341.html) (smaller). Mouser/DigiKey carry ILI9341 breakouts (Adafruit/Newhaven) but with different pinouts — **not drop-in**. |
| GY-PCM5102 DAC module | 1 | — | — | — | [AliExpress](https://www.aliexpress.us/item/3256802711963831.html) |
| Adafruit Micro SD SPI/SDIO Breakout | 1 | — | — | — | [Adafruit](https://www.adafruit.com/product/4682) |
| Kailh Choc V1 key switch | 9 | — | — | — | specialty (mechanical-keyboard vendors) |
| keycaps (Kailh Choc) | 9 | — | — | — | specialty |
| MAX98306 class-D amp module | 1 | — | — | — | [AliExpress](https://ja.aliexpress.com/item/1005004990814956.html) (Adafruit #987) |
| Micro speaker | 2 | — | — | — | [Akizuki](https://akizukidenshi.com/catalog/g/gP-12494/) |
| DPDT slide switch (through-hole) | 1 | ~$0.84 | [JS202011AQN](https://www.mouser.com/c/?q=JS202011AQN) | [JS202011AQN](https://www.digikey.com/en/products/detail/c-k/JS202011AQN/1640096) | J9 (KILL_SWITCH); C&K JS202011AQN, right-angle. Generic SS-22 / MSK22D18G2 (LCSC C2906279, ~$0.07) also fits. GND on center pins. |
| PJ-311 3.5mm stereo jack | 2 | varies | — | — | MIDI in/out (J1, J2). Generic PJ-311/PJ-320 import — no footprint-compatible Mouser/DigiKey catalog part. |
| 1N4148 | 1 | ~$0.10 | [1N4148](https://www.mouser.com/c/?q=1N4148) | [1N4148](https://www.digikey.com/en/products/result?keywords=1N4148) | D1; DO-35 axial |
| 47R | 2 | ~$0.11 | [CFR-25JB-52-47R](https://www.mouser.com/c/?q=CFR-25JB-52-47R) | [CFR-25JB-52-47R](https://www.digikey.com/en/products/result?keywords=CFR-25JB-52-47R) | axial 1/4W (DIN0207, 7.62mm) |
| 220R | 1 | ~$0.10 | [CFR-25JB-52-220R](https://www.mouser.com/c/?q=CFR-25JB-52-220R) | [CFR-25JB-52-220R](https://www.digikey.com/en/products/result?keywords=CFR-25JB-52-220R) | axial 1/4W (DIN0207, 7.62mm) |
| 470R | 1 | ~$0.11 | [CFR-25JB-52-470R](https://www.mouser.com/c/?q=CFR-25JB-52-470R) | [CFR-25JB-52-470R](https://www.digikey.com/en/products/result?keywords=CFR-25JB-52-470R) | axial 1/4W (DIN0207, 7.62mm) |
| 10k | 1 | ~$0.10 | [CFR-25JB-52-10K](https://www.mouser.com/c/?q=CFR-25JB-52-10K) | [CFR-25JB-52-10K](https://www.digikey.com/en/products/result?keywords=CFR-25JB-52-10K) | axial 1/4W (DIN0207, 7.62mm) |
| 6N138 | 1 | ~$0.83 | [6N138](https://www.mouser.com/c/?q=6N138) | [6N138](https://www.digikey.com/en/products/result?keywords=6N138) | U2; DIP-8 optocoupler |
| LiPo battery | 1 | — | — | — | 1200mAh (author's choice) — hobby battery vendors |
| Pin headers / standoffs / screws / nuts | * | — | [headers](https://www.mouser.com/c/?q=2.54mm+pin+header) | [headers](https://www.digikey.com/en/products/result?keywords=2.54mm+pin+header) | See note below on short headers vs sockets |

¹ **Avg unit price** — indicative qty-1 price in USD, from Mouser (the DPDT switch is averaged with DigiKey). Modules are priced at their linked vendors (they aren't stocked by Mouser/DigiKey). Most distributor cells are keyword/site searches — pick the exact package/stock on the results page; the DPDT switch uses direct product links (Mouser and DigiKey). Stock and prices change, so confirm at the link before ordering.

It is recommended to use short pin headers and to solder directly without using sockets. At the very least, the effective depth of the 3D printer back panel is only 11 mm, which would interfere with the use of pin sockets. (but i wanted to go thin)

The images below are prototype versions and differ from the actual data.

<img src="https://raw.githubusercontent.com/ijnekenamay/picotracker_alt-pcb/main/images/1.jpg" width="400"><img src="https://raw.githubusercontent.com/ijnekenamay/picotracker_alt-pcb/main/images/2.jpg" width="400">
