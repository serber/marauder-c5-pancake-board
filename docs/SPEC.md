# Marauder C5 Pancake Board — PCB specification

Evidence labels: **[fact]** checked against a source, **[decision]** a design choice, **[estimate]** to confirm by calculation or measurement.

## 1. Purpose

One board replaces the set of modules and wires in the Pancake build: DevKit, TP4056 with boost, MAX17048, and ATGM336H.

The board plugs directly onto the pin header of the 3.5" display. The board comes first, and a custom case is designed around it afterwards. It carries:

- ESP32-C5 and its support circuitry
- one USB-C port for flashing, console, and charging
- 18650 charging and protection
- a battery fuel gauge
- a header for an external GPS module

## 2. Findings

### 2.1 Pancake and the official V8 are different hardware

| | Marauder V8 (official) | Pancake (the case target) |
|---|---|---|
| Display | 2.8", 240×320, ILI9341 | 3.5", 320×480, ST7796 |
| Touch | Resistive (no `HAS_CAP_TOUCH`) | Capacitive FT6336 over I2C |
| I2C | GPIO5 (SDA) / GPIO4 (SCL) | GPIO9 (SDA) / GPIO10 (SCL) |
| SD_CS | GPIO10 | GPIO7 |
| Firmware | Protected `_v8.bin` | Public `pancake` build from CI |
| Third-party firmware | No data | JanOS, GhostESP, Launcher, and others listed on the case page |

**[fact]** The wiki warns that V8 and Pancake firmware are not interchangeable. CI has no public V8 build target, and the repository has no V8 display setup file. The Pancake build calls itself `"Pancake Marauder V8"`, which explains the naming confusion.

**[decision]** We copy the Pancake pinout exactly, so the firmware works without changes. The board uses the `MARAUDER_PANCAKE` target, an 8 MB module, and PSRAM enabled.

### 2.2 GPS occupies the ESP32-C5 USB pins

**[fact]** GPIO13 and GPIO14 are also `USB_D-` and `USB_D+` of the native USB. All Pancake firmware uses them for GPS. This is why the DevKit has two ports, `UART` and `USB`.

**[decision]** The single USB-C connects through a USB-UART bridge to UART0 (GPIO11/12), with automatic bootloader entry. Native USB stays unused. The Pancake build does not set `CDCOnBoot`, so the Marauder console is also on UART0, and the port serves both flashing and the CLI.

Moving GPS to other pins is not an option: every firmware would need a custom build.

### 2.3 Display

- **[fact]** The chosen module is labelled "3.5" TFT SPI 480X320 V1.0 Capacitive Touch". The header silkscreen in the user's image matches the lcdwiki MSP3526 schematic (ST7796U, FT6336U). The FPC legend in that image is garbled, and the image looks like a seller render, so the header silkscreen is the only part of it we rely on. Check the real module on arrival.
- **[fact]** The connector has 14 pins at 2.54 mm pitch, on a short edge. Size and hole data are in section 6.
- **[fact]** The microSD slot is **on the display module**, powered from the module's 3.3 V LDO. Our board needs no SD slot, only SD_CS to the connector.
- **[fact]** `VCC` feeds an XC6206 LDO, which makes `VCC3.3`. That rail powers the LCD, backlight anode, touch, level shifter, and SD card.
- **[fact]** The backlight is on a BSS138 low-side switch with a 2 Ω resistor. `LED` has a 10 kΩ pull-up to `VCC3.3`: high or open is on, low is off, and PWM works.
- **[fact]** `CTP_SDA` and `CTP_SCL` have 10 kΩ pull-ups to **`VCC`** on the header side.
- **[decision]** Feed `VCC` from the 3V3 rail, as the Pancake documentation does, with no 5 V rail. At `VCC` = 5 V, the I2C pull-ups would put 5 V on GPIO9/10, which are 3.3 V pins.
- **Consequence:** at `VCC` = 3.3 V the LDO works in dropout, so `VCC3.3` sits slightly below 3.3 V and the backlight may be dimmer than at 5 V. lcdwiki rates the module at 5 V. Measure brightness and SD stability at bring-up.
- **Mismatch:** the Pancake README and case BOM say ILI9341, while the Marauder setup and the lcdwiki drawing say ST7796. We follow Marauder and lcdwiki.

## 3. Requirements

### Functions

| ID | Requirement | Source |
|---|---|---|
| F1 | ESP32-C5 module with external antenna (U.FL), 8 MB flash, PSRAM | Brief, firmware |
| F2 | Pinout exactly as in section 4 | 2.1 |
| F3 | One USB-C: flashing, console, charging | Brief |
| F4 | One-click flashing: automatic bootloader entry and reset | Brief |
| F5 | RESET and BOOT buttons. BOOT also acts as the select button (GPIO28, `C_BTN`) | Firmware |
| F6 | Addressable LED on GPIO27, on the display side, seen through a side window in the case | Firmware, reviewer |
| F7 | MAX17048 fuel gauge on I2C, address 0x36, on the bus shared with touch | Firmware |
| F8 | ~~On-board GPS footprint~~ Removed by the reviewer on 2026-11-06 to free board area. The board works fully without GPS | Reviewer |
| F9 | External GPS header J2: filtered 3V3, GND, TX, RX, PPS. PPS is a free pin, not connected to the ESP | Brief |
| F10 | Passive buzzer on GPIO6 through a transistor | Case build |
| F11 | Display socket 1×14, 2.54 mm, position per section 6; the display plugs directly into it | 2.3, 6 |

### Power

| ID | Requirement | Source |
|---|---|---|
| P1 | Powered by one 18650 cell, 3.0–4.2 V | Brief |
| P2 | On-board charging from USB-C, 500 mA input, about 0.45 A into the cell **[decision]** | Brief |
| P3 | Operation while charging (power path), so charge termination is not disturbed by the load | **[decision]** |
| P4 | 3V3 rail from a buck-boost DC-DC, at least 1 A, see the budget | Section 5 |
| P5 | On-board cell protection: overcharge, overdischarge, short circuit, so a bare cell is safe | **[decision]** |
| P6 | Protection against reversed cell insertion, a real risk with any open holder | **[decision]** |
| P7 | Load cutoff on discharge at no lower than 3.0 V **[decision]**, with margin to the cell minimum | Cell |
| P8 | A power control fully turns off the 3V3 rail and all loads. Charging works while off. Only the cell protection and fuel gauge stay connected | User request, **[decision]** |
| P9 | Clean GPS supply: input ripple up to 50 mVpp, so a filter from the DC-DC is needed | ATGM336H manual |
| P10 | A powered-off USB bridge and a powered-off ESP do not back-power each other through the UART lines | **[decision]** |
| P11 | The cell is user-swappable. Charging outside the device is the normal use, and on-board charging is an extra. After a freshly charged cell is inserted, the board must start without USB | User request |

### Mechanics and manufacturing

| ID | Requirement | Source |
|---|---|---|
| M1 | The board outline is 55.50 × 98.00 mm, and the display mounting holes line up | 6 |
| M2 | USB-C, switch, buttons, and the SD slot stay reachable from the board edges in a future case | 6 |
| M3 | A 3D assembly with the cell and display is checked for clearances, and a STEP file is exported for the case design | Step 4 |
| M4 | 4 layers, 1.6 mm **[decision]**: solid ground under the DC-DC | — |
| M5 | Parts suit hand assembly on a hot table: 0603 passives where possible. Each part has datasheet and order links, and key parts must be buyable from more than one shop | User |
| M6 | Test points: VBAT, VSYS, 3V3, UART0, EN, GPIO28 | Debugging |

### Out of scope

On-board SD slot, native ESP32-C5 USB, on-board Wi-Fi antenna, certification.

## 4. Pinout

Pin numbers are for the ESP32-C5-WROOM-1U. GPIO assignments are checked against `configs.h` and `User_Setup_marauder_pancake.h` (Marauder v1.20.0).

| GPIO | Pin | Signal | Connects to | Notes |
|---|---|---|---|---|
| 2 | 4 | TFT_RST | Display 4 | Strapping (MTMS) |
| 3 | 5 | TFT_DC | Display 5 | Strapping (MTDI) |
| 4 | 17 | SPI MISO | Display 9 | Shared by LCD and SD |
| 5 | 16 | TFT_CS | Display 3 | |
| 6 | 8 | Buzzer | Transistor | BAT_ADC in JanOS, see below |
| 7 | 9 | SD_CS | Display 14 | Strapping (JTAG source) |
| 8 | 10 | CTP_RST | Display 11 | |
| 9 | 11 | I2C SDA | Display 12, MAX17048 | 10 kΩ pull-up on the display; board pads unpopulated |
| 10 | 12 | I2C SCL | Display 10, MAX17048 | 10 kΩ pull-up on the display; board pads unpopulated |
| 11 | 25 | U0TXD | USB-UART bridge | |
| 12 | 24 | U0RXD | USB-UART bridge | |
| 13 | 13 | GPS RX (ESP TX) | GPS RXD | Also USB_D-, unused |
| 14 | 14 | GPS TX (ESP RX) | GPS TXD | Also USB_D+, unused |
| 23 | 21 | SPI SCK | Display 7 | Shared by LCD and SD |
| 24 | 23 | SPI MOSI | Display 6 | Shared by LCD and SD |
| 25 | 26 | CTP_INT | Display 13 | Strapping |
| 26 | 27 | TFT_BL (PWM) | Display 8 | Strapping |
| 27 | 18 | LED | DIN | Strapping, must be 1 at boot |
| 28 | 15 | BOOT / button | Button, auto-bootloader | Strapping: 1 = run, 0 = bootloader |
| 0, 1 | 6, 7 | Free | Pads | Spare |

GPS direction is checked in code: `Serial2.begin(9600, SERIAL_8N1, GPS_TX=14, GPS_RX=13)`, so the ESP receives on GPIO14. Module TX goes to GPIO14, and module RX goes to GPIO13.

Display connector (lcdwiki schematic, matches the header silkscreen in the user's image):
`1 VCC · 2 GND · 3 LCD_CS · 4 LCD_RST · 5 LCD_RS · 6 SDI · 7 SCK · 8 LED · 9 SDO · 10 CTP_SCL · 11 CTP_RST · 12 CTP_SDA · 13 CTP_INT · 14 SD_CS`

Boot **[fact]**: GPIO28 = 1 boots from flash. GPIO27 = 1 with GPIO28 = 0 enters the bootloader. We add external pull-ups on GPIO27/28 and an RC on EN (10 kΩ, 1 µF, as the datasheet recommends).

GPIO6 **[decision]**: the buzzer is connected by default. Pads for a VBAT divider (200 kΩ/100 kΩ, as on the Waveshare board) stay unpopulated for JanOS. The buzzer and divider are not used at the same time.

## 5. Architecture

```
USB-C ─┬─ ESD ─ D+/D- ─ USB-UART bridge ─ UART0 + auto-bootloader ─ ESP32-C5
       │
       └─ VBUS ─ charger with power path ─ VSYS ─ switch ─ DC-DC 3V3 ─┬─ ESP32-C5
                       │                                              ├─ display (SD on it)
       18650 ─ protection ─┴─ MAX17048 (I2C)                          ├─ LED, buzzer
                                                                      └─ filter ─ GPS (optional)
```

3V3 rail budget:

| Load | Peak, mA | Basis |
|---|---|---|
| ESP32-C5, 5 GHz TX | 403 | [fact] Datasheet, table 6-5 |
| Backlight | 95 | [fact] lcdwiki |
| Display logic, touch | 20 | [estimate] |
| SD card | 100 | [estimate] |
| GPS | 30 | [estimate] |
| Bridge, LED, buzzer | 30 | [estimate] |
| **Total** | **~680** | Design for 1 A |

At 3.0 V on the cell and about 90 % efficiency **[estimate]**, cell current reaches about 1.2 A.

## 6. Mechanics

Values come from the [MSP3526 outline drawing](https://www.lcdwiki.com/res/MSP3525_MSP3526/3.5inch_SPI_Module_MSP3526_Size.pdf) (rev. V1.0, 2023-04-15) unless marked otherwise. Check them against the real display and holder before ordering.

### Coordinate system

All board coordinates use one view: **looking at the electronics side of our board, which is also looking at the rear of the display**. The display socket is at the bottom.

- Origin: bottom-left corner of the board
- +X: right, +Y: up
- Units: mm

In EasyEDA, the electronics side is the **top layer**. The display socket is a through-hole part fitted on the **bottom layer** side.

### Board outline

| Item | Value | Evidence |
|---|---|---|
| Outline | 55.50 × 98.00 mm | [fact] matches the display PCB |
| Thickness | 1.6 mm | [decision] |
| Layers | 4 | [decision] |

| Hole | X | Y |
|---|---|---|
| H1 | 3.00 | 3.00 |
| H2 | 52.50 | 3.00 |
| H3 | 3.00 | 95.00 |
| H4 | 52.50 | 95.00 |

Holes are Ø3.20 mm for M3, with a 49.50 × 92.00 mm pattern **[fact]**. Pads on our board are Ø5.0 mm on GND, reduced from Ø5.5 mm to meet the outline clearance **[decision]**. Corners have a 3.0 mm radius, concentric with the holes **[decision]**.

### Display socket

The socket is a 1×14 female header, 2.54 mm pitch, on the display-facing side. Pins sit at Y = 2.00.

Pin n sits at X = 44.26 − (n − 1) × 2.54, so pin 1 (VCC) is on the right at X = 44.26 and pin 14 (SD_CS) is on the left at X = 11.24 **[fact]**.

| Pin | Signal | X | | Pin | Signal | X |
|---|---|---|---|---|---|---|
| 1 | VCC | 44.26 | | 8 | LED | 26.48 |
| 2 | GND | 41.72 | | 9 | SDO (MISO) | 23.94 |
| 3 | LCD_CS | 39.18 | | 10 | CTP_SCL | 21.40 |
| 4 | LCD_RST | 36.64 | | 11 | CTP_RST | 18.86 |
| 5 | LCD_RS | 34.10 | | 12 | CTP_SDA | 16.32 |
| 6 | SDI (MOSI) | 31.56 | | 13 | CTP_INT | 13.78 |
| 7 | SCK | 29.02 | | 14 | SD_CS | 11.24 |

### Stack height

| Layer, front to back | Height, mm | Evidence |
|---|---|---|
| Display without pins | 5.90 | [fact] |
| Male header plastic | 2.54 | [fact] |
| Female socket | 8.5 | [estimate] standard part, check the real one |
| Board | 1.6 | [decision] |
| Holder with cell | about 20 | [estimate] holder 14.86 mm from the Keystone 1042 listing |
| **Total** | **about 38.5** | [estimate] |

The gap between the display PCB and our board is about 11.0 mm, so M3 × 11 mm standoffs fit **[estimate]**. Confirm the gap with the real socket.

The display pins are 5.84 mm long above their plastic **[fact]**. A socket lower than 8.5 mm needs a cavity at least that deep, or trimmed pins.

### Sides of the board

**[decision]** All SMD parts and the holder go on the rear (top layer). The display-facing side carries the socket, standoffs, LED1, and C14.

Reasons:

- One reflow pass on the hot table.
- Nothing on the display side can touch the SD slot, the card, or the FPC.
- Every part stays reachable for probing, with buttons, USB-C, and U.FL exposed.

The cell holder is fitted by hand after reflow, because the plastic of an unbranded holder is not known to survive reflow.

### Holder

| Item | Value | Evidence |
|---|---|---|
| Body | 77 × 20.8 mm | [fact] seller image |
| Height | 14.86 mm | [fact] Keystone 1042 listing, not the bought part |
| Tab-to-tab length | Footprint pad centers 78.7 mm, span 86.7 mm | [fact] for Keystone 1042 footprint. Measure the bought part |

**[decision]** The body lies along the long axis, against the left edge, centered in Y.

| Item | X | Y |
|---|---|---|
| Body | 0.5 to 21.3 | 10.5 to 87.5 |
| Tabs | About 8 to 14 | Beyond the body ends, up to about Y 5 and Y 93 [estimate] |

This clears holes H1 and H3 and the socket pins [estimate]. The electronics zone is X 22.3 to 55.5, 33.2 mm wide, with the full 98 mm height minus the holes and socket.

The left/right choice can still change in step 6.

The cell is user-swappable (P11), so the future case needs a battery door or a removable back, and the holder stays open from the rear.

### Keep-outs

| Zone | Rule | Reason |
|---|---|---|
| Under the holder body | No parts, no exposed copper, no vias on top | Open frame, and the can sits close to the board |
| Display-facing side | Only the socket, standoffs, LED1, and C14 | SD card, FPC, display parts up to 2.20 mm |
| Around holes | Ø5.5 mm free of parts on both sides; only GND copper is exposed, on the Ø5.0 mm pad | M3 hardware. Non-GND tracks under solder mask come as close as 2.76 mm from the center of H4 |
| Right edge near Y 51 | Keep the edge accessible in the case | SD card opening on the display |

### SD card access

The slot opens toward the right long edge in our view (+X), with its notch center about 51 mm from the socket edge, Y ≈ 50.97 **[fact]** (47.03 mm from the opposite short edge). The case needs a slot there, in the display plane. The board does not block it.

### Connectors and controls

Exact positions are set in step 6. All go on the electronics side.

| Part | Edge | Note |
|---|---|---|
| USB-C | Right long edge, Y ≈ 83.6, beside SW1 | Moved by the reviewer |
| Slide switch | Right long edge, Y ≈ 72 | Sets the case opening |
| RESET, BOOT | Rear, near the bottom edge | Need a push rod or an opening in the case |
| U.FL (Wi-Fi) | On the module | Pigtail to the SMA in the case |
| External GPS header | J2, rear, X ≈ 45.5, Y ≈ 19.5 | 1×5, 2.54 mm |
| LED1 | Display side, (45.5, 94.4), top short edge | Needs a side window in the case. The display PCB covers it from the front |

### Mechanical open items

1. Measure the real display: holes, socket position, and pin order.
2. Measure the real holder: tab spacing, tab size, height, and cell-to-board distance.
3. Measure the real socket height.

For the future case, step 9 exports a STEP model of the assembly.

## 7. Sources

Checked 2026-11-05:

- [Marauder v8 wiki](https://github.com/justcallmekoko/ESP32Marauder/wiki/marauder-v8)
- [ESP32Marauder source](https://github.com/justcallmekoko/ESP32Marauder), commit `511bb36`: `esp32_marauder/configs.h`, `User_Setup_marauder_pancake.h`, `GpsInterface.cpp`, `.github/workflows/build_parallel.yml`
- [Battery Fuel Gauge wiki](https://github.com/justcallmekoko/ESP32Marauder/wiki/battery-fuel-gauge), [GPS Modification wiki](https://github.com/justcallmekoko/ESP32Marauder/wiki/gps-modification)
- [C5Lab Pancake case on MakerWorld](https://makerworld.com/ru/models/2662148-c5lab-pancake-esp32-c5-case#profileId-2944904)
- [C5Lab/pancake](https://github.com/C5Lab/pancake): `README.md`, `ESP32C5/screen_connections_readme.md`
- [ESP32-C5-WROOM-1/1U datasheet v1.3](https://documentation.espressif.com/esp32-c5-wroom-1_wroom-1u_datasheet_en.pdf)
- [ESP32-C5 datasheet](https://documentation.espressif.com/esp32-c5_datasheet_en.pdf)
- [lcdwiki, 3.5" SPI module ST7796 (MSP3525/MSP3526)](https://www.lcdwiki.com/3.5inch_IPS_SPI_Module_ST7796)
- [ATGM336H-5N manual](https://wiki.millerjs.org/_media/atgm336h-5n_datasheet.pdf)
- [MSP3526 outline drawing](https://www.lcdwiki.com/res/MSP3525_MSP3526/3.5inch_SPI_Module_MSP3526_Size.pdf), [MSP3525/MSP3526 schematic](https://www.lcdwiki.com/res/MSP3525_MSP3526/3.5inch_SPI_Module_MSP3525_MSP3526_Schematic.pdf)
- [Keystone 1042 listing](https://www.futureelectronics.com/p/6068466)
- EasyEDA Pro library: LCSC codes. Stock and prices are not yet checked.
- [TPS63802 datasheet, SLVSEU9D](https://www.ti.com/lit/ds/symlink/tps63802.pdf)
- [BQ2407x datasheet, SLUS810N](https://www.ti.com/lit/ds/symlink/bq24074.pdf)
- [ESP32-C5-WROOM-1/1U datasheet v1.3](https://documentation.espressif.com/esp32-c5-wroom-1_wroom-1u_datasheet_en.pdf)
- [ATGM336H-5N manual](https://wiki.millerjs.org/_media/atgm336h-5n_datasheet.pdf)
