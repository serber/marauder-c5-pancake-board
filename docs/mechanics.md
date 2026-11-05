# Mechanics (step 1a)

Status: ready for review. Values come from the [MSP3526 outline drawing](https://www.lcdwiki.com/res/MSP3525_MSP3526/3.5inch_SPI_Module_MSP3526_Size.pdf) (rev. V1.0, 2023-04-15) unless marked otherwise. Check them against the real display and holder before ordering boards.

Evidence labels match `requirements-and-plan.md`: **[fact]**, **[decision]**, **[estimate]**.

## Coordinate system

All board coordinates use one view: **looking at the electronics side of our board, which is also looking at the rear of the display**. The display socket is at the bottom.

- Origin: bottom-left corner of the board
- +X: right, +Y: up
- Units: mm

In EasyEDA, the electronics side is the **top layer**. The display socket is a through-hole part fitted on the **bottom layer** side.

## Board outline

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

Holes are Ø3.20 mm for M3, with a 49.50 × 92.00 mm pattern **[fact]**. Pads on our board are Ø5.5 mm on GND **[decision]**. Corners have a 3.0 mm radius, concentric with the holes **[decision]**.

## Display socket

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

## Stack height

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

## Sides of the board

**[decision]** All SMD parts and the holder go on the rear (top layer). The display-facing side carries the socket, standoffs, LED1, and C14.

Reasons:

- One reflow pass on the hot table.
- Nothing on the display side can touch the SD slot, the card, or the FPC.
- Every part stays reachable for probing, with buttons, USB-C, and U.FL exposed.

The cell holder is fitted by hand after reflow, because the plastic of an unbranded holder is not known to survive reflow.

## Holder

| Item | Value | Evidence |
|---|---|---|
| Body | 77 × 20.8 mm | [fact] seller image |
| Height | 14.86 mm | [fact] Keystone 1042 listing, not the bought part |
| Tab-to-tab length | Unknown | Measure |

**[decision]** The body lies along the long axis, against the left edge, centered in Y.

| Item | X | Y |
|---|---|---|
| Body | 0.5 to 21.3 | 10.5 to 87.5 |
| Tabs | About 8 to 14 | Beyond the body ends, up to about Y 5 and Y 93 [estimate] |

This clears holes H1 and H3 and the socket pins [estimate]. The electronics zone is X 22.3 to 55.5, 33.2 mm wide, with the full 98 mm height minus the holes and socket.

The left/right choice can still change in step 6.

The cell is user-swappable (P11), so the future case needs a battery door or a removable back, and the holder stays open from the rear.

## Keep-outs

| Zone | Rule | Reason |
|---|---|---|
| Under the holder body | No parts, no exposed copper, no vias on top | Open frame, and the can sits close to the board |
| Display-facing side | Only the socket, standoffs, LED1, and C14 | SD card, FPC, display parts up to 2.20 mm |
| Around holes | Ø5.5 mm free on both sides | M3 hardware |
| Right edge near Y 51 | Keep the edge accessible in the case | SD card opening on the display |

## SD card access

The slot opens toward the right long edge in our view (+X), with its notch center about 51 mm from the socket edge, Y ≈ 50.97 **[fact]** (47.03 mm from the opposite short edge). The case needs a slot there, in the display plane. The board does not block it.

## Connectors and controls

Exact positions are set in step 6. All go on the electronics side.

| Part | Edge | Note |
|---|---|---|
| USB-C | Right long edge, Y ≈ 83.6, beside SW1 | Moved by the reviewer |
| Slide switch | Right long edge, Y ≈ 72 | Sets the case opening |
| RESET, BOOT | Rear, near the bottom edge | Need a push rod or an opening in the case |
| U.FL (Wi-Fi) | On the module | Pigtail to the SMA in the case |
| External GPS header | J2, rear, X ≈ 45.5, Y ≈ 19.5 | 1×5, 2.54 mm |
| LED1 | Display side, (45.5, 94.4), top short edge | Needs a side window in the case. The display PCB covers it from the front |

## Open items

1. Measure the real display: holes, socket position, and pin order.
2. Measure the real holder: tab spacing, tab size, height, and cell-to-board distance.
3. Measure the real socket height.

For the future case, step 9 exports a STEP model of the assembly.
