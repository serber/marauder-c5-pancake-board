# Marauder ESP32-C5 board with the Pancake pinout: requirements and plan

Status: **steps 1 to 9 done; remaining gates waived by the reviewer on 2026-11-06. Prototype package exported, not ordered, board not validated**
Date: 2026-11-06 · Board revision: rev1 (prototype, not built)

This is the main project document. All work follows section 8, and decisions go in the log in section 10.

Evidence labels:
- **[fact]**: checked against a source (section 11)
- **[decision]**: a proposed choice that can still change
- **[estimate]**: a rough figure to confirm by calculation or measurement

---

## 1. What we are building

One board replaces the set of modules and wires in the Pancake build: DevKit, TP4056 with boost, MAX17048, and ATGM336H.

The board plugs directly onto the pin header of the 3.5" display. The board comes first, and a custom case is designed around it afterwards. It carries:

- ESP32-C5 and its support circuitry
- one USB-C port for flashing, console, and charging
- 18650 charging and protection
- a battery fuel gauge
- a header for an external GPS module

---

## 2. Open questions

| ID | Question | Proposal | Affects |
|---|---|---|---|
| ~~Q1~~ | ~~Is the target the Pancake pinout, not the official V8?~~ | **Closed:** Pancake, see section 10 | Whole schematic |
| ~~Q2~~ | ~~Can we get the case files?~~ | **Closed:** no existing case, see section 10 | Board outline, height, mounting |
| ~~Q3~~ | ~~Which display will be used?~~ | **Closed:** the "3.5" TFT SPI 480X320 V1.0 Capacitive Touch" module, matched to lcdwiki MSP3526, see 3.3 | Display connector and power |
| ~~Q4~~ | ~~JLCPCB PCBA or by hand?~~ | **Closed:** hand assembly by the user | Packages, BOM |
| ~~Q5~~ | ~~Which case opening should the USB-C use?~~ | **Closed:** the board sets connector positions, see 3.4 | Component placement |
| ~~Q6~~ | ~~Which exact holder?~~ | **Closed:** design for the Keystone 1042 size class, user buys a matching holder, see 3.4 | Footprint, rear layout, thickness |
| ~~Q7~~ | ~~What hand-assembly tools are available?~~ | **Closed:** SMD experience, hot table, iron; parts from several shops including AliExpress | Packages, BOM |
| ~~Q8~~ | ~~Slide switch or latching button?~~ | **Closed:** slide switch on the board edge, driving DC-DC enable | P8, board edge |
| ~~Q9~~ | ~~USB input current?~~ | **Closed:** fixed 500 mA input | Charge time, compatibility with weak USB ports |

---

## 3. Findings

### 3.1 Pancake and the official V8 are different hardware

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

### 3.2 GPS occupies the ESP32-C5 USB pins

**[fact]** GPIO13 and GPIO14 are also `USB_D-` and `USB_D+` of the native USB. All Pancake firmware uses them for GPS. This is why the DevKit has two ports, `UART` and `USB`.

**[decision]** The single USB-C connects through a USB-UART bridge to UART0 (GPIO11/12), with automatic bootloader entry. Native USB stays unused. The Pancake build does not set `CDCOnBoot`, so the Marauder console is also on UART0, and the port serves both flashing and the CLI.

Moving GPS to other pins is not an option: every firmware would need a custom build.

### 3.3 Display

- **[fact]** The chosen module is labelled "3.5" TFT SPI 480X320 V1.0 Capacitive Touch". The header silkscreen in the user's image matches the lcdwiki MSP3526 schematic (ST7796U, FT6336U). The FPC legend in that image is garbled, and the image looks like a seller render, so the header silkscreen is the only part of it we rely on. Check the real module on arrival.
- **[fact]** The connector has 14 pins at 2.54 mm pitch, on a short edge. Size and hole data are in 3.4.
- **[fact]** The microSD slot is **on the display module**, powered from the module's 3.3 V LDO. Our board needs no SD slot, only SD_CS to the connector.
- **[fact]** `VCC` feeds an XC6206 LDO, which makes `VCC3.3`. That rail powers the LCD, backlight anode, touch, level shifter, and SD card.
- **[fact]** The backlight is on a BSS138 low-side switch with a 2 Ω resistor. `LED` has a 10 kΩ pull-up to `VCC3.3`: high or open is on, low is off, and PWM works.
- **[fact]** `CTP_SDA` and `CTP_SCL` have 10 kΩ pull-ups to **`VCC`** on the header side.
- **[decision]** Feed `VCC` from the 3V3 rail, as the Pancake documentation does, with no 5 V rail. At `VCC` = 5 V, the I2C pull-ups would put 5 V on GPIO9/10, which are 3.3 V pins.
- **Consequence:** at `VCC` = 3.3 V the LDO works in dropout, so `VCC3.3` sits slightly below 3.3 V and the backlight may be dimmer than at 5 V. lcdwiki rates the module at 5 V. Measure brightness and SD stability at bring-up.
- **Mismatch:** the Pancake README and case BOM say ILI9341, while the Marauder setup and the lcdwiki drawing say ST7796. We follow Marauder and lcdwiki.

### 3.4 Mechanics

There is no existing case. The board comes first, and a custom case follows, so the board sets the geometry. The [C5Lab Pancake case](https://makerworld.com/ru/models/2662148-c5lab-pancake-esp32-c5-case#profileId-2944904) is a layout reference only.

Display data from the lcdwiki MSP3526 drawing **[fact]**:

| Item | Value |
|---|---|
| PCB | 55.50 × 98.00 mm |
| Mounting holes | 4 × Ø3.20 mm, pads Ø4.70 mm, spaced 49.50 × 92.00 mm, 3.00 mm from the edges |
| Header | 14 pins, centered on a short edge: 11.24 mm from each long edge, pin row 2.00 mm from the short edge |
| Thickness without pins | 5.90 mm |
| Rear SMD height | 2.20 mm max |
| Header height from the rear of the display PCB | 8.38 mm (plastic 2.54 mm, pin 5.84 mm) |

**[decision]** Stack, front to back: display, 2.54 mm socket and standoffs, board, cell in the holder on the rear.

- The outline matches the display (55.50 × 98.00 mm), with the same four holes. Standoffs at the holes take the load.
- A standard 8.5 mm socket gives a gap of about 11 mm between the boards **[estimate]**.
- Board pin 1 must be checked in 3D: the board faces the display rear, so the header order mirrors.
- Parts on the display-facing side must clear the display rear and the path of the microSD card.
- USB-C, the power switch, and buttons sit on the board edges.
- Nothing tall, and no exposed copper or vias, goes under the holder: the bottom of the frame is open, and the can is at cell negative.
- The U.FL pigtail runs to an SMA fitted in the future case.

**[decision]** The holder is an open-frame SMT type soldered by its tabs, in the Keystone 1042 size class: listed at 77.05 × 20.65 × 14.86 mm. EasyEDA has this footprint (C5249310). The holder found by the user has a 77 × 20.8 mm body, which matches this class. The seller gives no tab-to-tab length or height, so measure the real part and check the footprint before ordering boards.

**[estimate]** Total thickness is about 38 mm: display 5.9, gap 11, board 1.6, holder with cell about 20.

---

## 4. Requirements

### Functions

| ID | Requirement | Source |
|---|---|---|
| F1 | ESP32-C5 module with external antenna (U.FL), 8 MB flash, PSRAM | Brief, firmware |
| F2 | Pinout exactly as in section 5 | 3.1 |
| F3 | One USB-C: flashing, console, charging | Brief |
| F4 | One-click flashing: automatic bootloader entry and reset | Brief |
| F5 | RESET and BOOT buttons. BOOT also acts as the select button (GPIO28, `C_BTN`) | Firmware |
| F6 | Addressable LED on GPIO27, on the display side, seen through a side window in the case | Firmware, reviewer |
| F7 | MAX17048 fuel gauge on I2C, address 0x36, on the bus shared with touch | Firmware |
| F8 | ~~On-board GPS footprint~~ Removed by the reviewer on 2026-11-06 to free board area. The board works fully without GPS | Reviewer |
| F9 | External GPS header J2: filtered 3V3, GND, TX, RX, PPS. PPS is a free pin, not connected to the ESP | Brief |
| F10 | Passive buzzer on GPIO6 through a transistor | Case build |
| F11 | Display socket 1×14, 2.54 mm, position per 3.4; the display plugs directly into it | 3.3, 3.4 |

### Power

| ID | Requirement | Source |
|---|---|---|
| P1 | Powered by one 18650 cell, 3.0–4.2 V | Brief |
| P2 | On-board charging from USB-C, 500 mA input, about 0.45 A into the cell **[decision]** | Brief, Q9 |
| P3 | Operation while charging (power path), so charge termination is not disturbed by the load | **[decision]** |
| P4 | 3V3 rail from a buck-boost DC-DC, at least 1 A, see the budget | Section 6 |
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
| M1 | The board outline is 55.50 × 98.00 mm, and the display mounting holes line up | 3.4 |
| M2 | USB-C, switch, buttons, and the SD slot stay reachable from the board edges in a future case | 3.4 |
| M3 | A 3D assembly with the cell and display is checked for clearances, and a STEP file is exported for the case design | Skill, step 4 |
| M4 | 4 layers, 1.6 mm **[decision]**: solid ground under the DC-DC | Section 7 |
| M5 | Parts suit hand assembly on a hot table: 0603 passives where possible. Each part has datasheet and order links, and key parts must be buyable from more than one shop | Q4, Q7, skill step 4 |
| M6 | Test points: VBAT, VSYS, 3V3, UART0, EN, GPIO28 | Debugging |

### Out of scope

On-board SD slot, native ESP32-C5 USB, on-board Wi-Fi antenna, certification.

---

## 5. Pinout

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

---

## 6. Preliminary architecture

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

Starting candidates from step 1. Final parts are in `circuit-design.md`.

| Block | Candidates | LCSC |
|---|---|---|
| Module | ESP32-C5-WROOM-1U-N8R8 (a `-V1.2` variant exists) | C51950748, C54951858 |
| Charger with power path | BQ24074RGTR, MCP73871 | C54313, C623875 |
| Buck-boost DC-DC | TPS63802, TPS63020, TPS631000 | C704850, C15483, C5219190 |
| Cell protection | DW01A + FS8205A, FS312F-G | C85680, C14212, C82736 |
| Fuel gauge | MAX17048G+T10 | C2682616 |
| USB-UART | CH343P, CP2102N-A02-GQFN20 | C2846043, C1550542 |
| USB ESD | USBLC6-2SC6 | C323793 |
| LED | XL-1615RGBC-2812B (check operation at 3.3 V) | C5349954 |
| Cell holder | Keystone 1042 size class (Q6). Alternatives: MY-18650-02, BH-18650-B1BA002 | C5249310, C2979182, C2988620 |

The GPS module is external, on J2.

---

## 7. Tools and skills

| Tool | Runs on | Version | Status |
|---|---|---|---|
| EasyEDA Pro | User's computer | 3.2.209 | Bridge on port 49620, window connected |
| EasyEDA project | `Marauder C5 Pancake` (recreated) | Board1: schematic1, PCB1 | Open, contents not yet changed |
| `easyeda-api` skill | Claude | — | Connected |
| `power-pcb-design` skill | `~/.claude/skills/power-pcb-design` | Copy from `.codex` | Added `references/easyeda-pro.md` |

How we apply `power-pcb-design`:

- Scope is **end-to-end, steps 1–9**, stopping for approval after each step.
- Loop analysis and the net table cover only the DC-DC, charger, and battery path.
- The whole board stays at extra-low voltage, so we skip insulation derivation and record that at step 5.
- No SPICE models are available yet. Anything not calculated or simulated is recorded as "no data". An estimate is never presented as a check.

Working through the bridge carries a risk: the project is open live and edits apply immediately. The build scripts and the exported netlist check let us rebuild and verify the schematic.

---

## 8. Work plan

Each step ends with a short report and approval (`approve` / `approve with changes` / `redo` / `stop`). The next step does not start without it.

| Step | Work | Result | Needed from the reviewer |
|---|---|---|---|
| **1. Requirements** | Questions closed | This file approved | Approval |
| **1a. Mechanics** | Check the display drawing against the real module, fix the holder and stack height, mark the keep-outs | `docs/mechanics.md`, board outline, height section | Approval |
| **2. Architecture** | Currents, DC-DC and charger loops, power modes, net table | `docs/power-design.md`, acceptance criteria | Approval |
| **3. Circuit on paper** | Part selection and calculations, LCSC stock, BOM cost | Part list with calculations | Approval |
| **4. Schematic in EasyEDA** | One sheet per function (section 8.1), ERC, link table, footprint audit, 3D models | Schematic in the project | Approval |
| **5. Rules** | Stack-up, net classes, DRC for JLCPCB, track widths | Rule table, EasyEDA settings | Approval |
| **6. Placement** | Connectors and mechanics first, then blocks: power, ESP, bridge, GPS | Image, metrics, 3D check | Approval |
| **7. Routing** | Routing plan, choice of mode A/B/C, routing by block | Routed board, DRC with no unexplained errors | Mode choice, block approvals |
| **8. Verification** | Current bottlenecks, return paths, thermal, check against requirements | Check record | Approval |
| **9. Release** | Gerber, stencil files, BOM, assembly drawing, STEP model for the case | File package. **Ordering only with permission** | Permission |
| **10. Bring-up** | First board per the checklist below, changes for revision B | Bring-up report | Assembly and measurements |

### 8.1 Schematic sheets

The user requires the schematic to be split into separate sheets by function. Each part also carries its group tag for step 6.

| Sheet | Contents |
|---|---|
| 01_Power_Input | USB-C, VBUS protection, charger |
| 02_Battery | Holder, cell protection, reverse protection, MAX17048 |
| 03_Regulator_3V3 | Switch, EN divider, TPS63802 stage, test points |
| 04_MCU | ESP32-C5 module, decoupling, EN, strapping, buttons, LED, buzzer |
| 05_USB_UART | USB-UART bridge, auto-bootloader |
| 06_Display | Display socket, I2C pull-up pads |
| 07_GPS | Supply filter, external GPS header |

### 8.2 First-board checklist


1. Without the ESP: power rails, quiescent current, switch. Insert a charged cell without USB 10 times (A13).
2. Protection: reversed cell, discharge and charge thresholds, short circuit.
3. Charging: with the device off and on.
4. USB: port detection, auto-bootloader, flashing `pancake`.
5. Display, touch, SD, battery percentage, LED, buzzer, button.
6. Wi-Fi 2.4/5 GHz, BLE, GPS through J2, and without GPS.
7. Heating and voltage droop during TX peaks at low charge.
8. Fit of the display and the cell holder; STEP check for the case.

---

## 9. Main risks

| Risk | Consequence | Mitigation |
|---|---|---|
| Holder or tall parts collide with the display or each other | Board rework | Step 1a before the schematic; 3D check |
| Different display revision: pin order or controller | Board does not work | Check the real module before ordering boards |
| Dim backlight or unstable SD at `VCC` = 3.3 V | Poor brightness | Measure at bring-up; raising the rail within the ESP32-C5 limit is the fallback |
| Exposed cell near the display-side parts or a hand-assembly short | Damage | Protection on board, test before fitting the cell |
| DC-DC noise reaches an external GPS | No fix | Filter on J2, module and antenna away from the DC-DC in the case |
| A part is out of stock | Delay | Two candidates for each key block |
| A Pancake update changes pins | Incompatibility | Version fixed at v1.20.0; recheck before ordering |
| 3V3 droop during TX peaks | Resets | Current margin, capacitors at the module, measurement |

---

## 10. Decision log

| Step | Date | Decision | Changes |
|---|---|---|---|
| 1 | 2026-11-05 | Q1: the target is the Pancake pinout with `pancake` firmware. The user confirmed by renaming the repository and recreating the project | Repository `marauder-c5-pancake-board`, EasyEDA project `Marauder C5 Pancake` |
| 1 | 2026-11-05 | All project files and Markdown are written in English | This file translated |
| 6–9 | 2026-11-06 | **Gates waived:** the reviewer approved all remaining steps in advance, unless something critical appears, and left the work unattended. Routing mode C chosen by the agent. No ordering, no push | `routing.md`, `verification.md`, `release/rev1/` |
| 4, 6 | 2026-11-06 | LED changed to SK6812MINI-HS on the display side, powered from DCDC_SW, with a side window in the case. USB-C moved to the right edge beside SW1. Parts kept away from the edge and holder | `placement.md`, `circuit-design.md`, `mechanics.md` |
| 6 | 2026-11-06 | Header kept for the display instead of the P2 flat cable | — |
| 1, 4, 5, 6 | 2026-11-06 | On-board GPS removed; J2 kept and populated. U8, J6, L3, R29 deleted, RF_50R class deleted, module moved down, placement respaced | All docs |
| 6 | 2026-11-06 | M3 holes and 3 mm rounded corners, per the reviewer. Placement done | `placement.md` |
| 5 | 2026-11-06 | Step 5 approved | — |
| 5 | 2026-11-06 | Stack-up, net classes, and rules set; four-layer template could not be applied through the API, so a customized two-layer template is used | `design-rules.md` |
| 4 | 2026-11-06 | Step 4 approved, including the bridge core on VBUS. Name-attribute ERC warnings accepted | — |
| 4 | 2026-11-06 | Schematic drawn on seven sheets. Bridge core moved from 3V3 to VBUS, per the CH343 datasheet; this changes the approach approved in step 2 and is up for review. Module changed to the V1.2 listing | `circuit-design.md`, `component-links.md`, `power-design.md` |
| 3 | 2026-11-06 | Step 3 approved | — |
| 3 | 2026-11-06 | A6 limit raised to 0.35 V. No pre-order DW01A test; cell start-up is checked on the first board | `power-design.md`, `circuit-design.md` |
| 3 | 2026-11-06 | Reverse protection is a low-side N-FET. Parts and values proposed in `circuit-design.md`; no simulation ran | `circuit-design.md`, `power-design.md` |
| 1a, 2 | 2026-11-06 | Steps 1a and 2 approved with Q9 option B | — |
| 1a, 2 | 2026-11-06 | Cell is swappable; on-board charging is an extra, so slow charging is acceptable | P11 |
| 1a, 2 | 2026-11-06 | Q9: fixed 500 mA USB input. Changes approved P2 from up to 1 A | Section 4, `power-design.md` |
| 1a, 2 | 2026-11-06 | Reviewer asked for one EasyEDA sheet per function | Section 8.1 |
| 1 | 2026-11-05 | Step 1 approved by the reviewer, gate log started | — |
| 1 | 2026-11-05 | Q6: Keystone 1042 size-class holder. Q7: hot-table hand assembly, mixed shops. Q8: slide switch on DC-DC enable | Sections 2, 3.4, 4, 6 |
| 1 | 2026-11-05 | Q3: display chosen, `VCC` from 3V3, no 5 V rail. Q4: hand assembly by the user. The power control fully turns off the system | Sections 3.3, 3.4, 4, 6, 8, 9 |
| 1 | 2026-11-05 | Q2, Q5: no existing case. The board comes first and a custom case follows. The Pancake case is a layout reference only | Sections 1, 2, 3.4, 4, 8, 9 |

---

## 11. Sources

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
