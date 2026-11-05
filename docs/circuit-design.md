# Circuit design (step 3)

Status: ready for review. Evidence labels match `power-design.md`: **DS** (datasheet), **Calc**, **Est**, **No data**.

Sheet numbers refer to `requirements-and-plan.md`, section 8.1.

## Simulation

No simulation ran. No SPICE tool is installed in this environment, and no DC-DC model has been obtained. The loop-inductance budgets in `power-design.md` therefore stay open. The layout follows the datasheet guidance instead, and bench measurement replaces simulation in `requirements-and-plan.md`, section 8.2.

## 01_Power_Input

| Item | Value | Basis |
|---|---|---|
| Input mode | USB500: EN1 to OUT, EN2 to GND | Q9; 450–500 mA (DS) |
| R_ISET | 1.96 kΩ 1 % | 890 / 1.96 k = 0.45 A, range 0.41–0.50 A (Calc) |
| Pre-charge | About 45 mA | 88 / 1.96 k (Calc) |
| Termination | ITERM open, default 10 %, about 45 mA | DS |
| R_ILIM | 3.3 kΩ | Inactive in USB500; keeps the pin defined in the 1.1–8 kΩ range (DS) |
| R_TMR | 71.5 kΩ | Fast-charge timer 9.5 h typ, 7.2–11.9 h (Calc) |
| TS | 10 kΩ to VSS | No NTC; the DS requires this resistor when TS is unused |
| CE | GND | Charging enabled |
| C_IN (charger) | 1 µF, 25 V | Rated for the input |
| C_OUT (charger) | 10 µF | 4.7–47 µF (DS) |
| C_BAT | 10 µF | 4.7–47 µF (DS) |
| CC1, CC2 | 5.1 kΩ each | USB-C sink |
| ESD | USBLC6-2SC6 on D+/D− | |
| Status LEDs | CHG and PGOOD, 1 kΩ to VBUS | Lit only with USB connected |

**Timer limit:** a full charge of 3500 mAh from empty needs 7.7 h typ, or 8.6 h at the low end of ICHG (Calc). At the shortest timer tolerance, charging can stop early with a fault. Re-plugging USB resumes it. A 2000–3200 mAh cell is covered.

## 02_Battery

Cell path:

```
Cell+ ── BAT node
Cell− ── Q_REV (D) ... Q_REV (S) ── B− ── FS8205A (two FETs) ── GND
```

| Item | Value | Basis |
|---|---|---|
| Q_REV | AO3400A N-MOSFET, drain to Cell−, source to B−, gate to Cell+ through 10 kΩ | Blocks a reversed cell with one series break |
| Protection | DW01A + FS8205A | R1 100 Ω and 0.1 µF on VDD; 1 kΩ on the sense pin |
| Bypass | Solder jumper across FS8205A, open by default | See risk below |
| Discharge overcurrent | About 2.5 A | 0.15 V / 60 mΩ (Est, DW01A variant in the cited datasheet) |
| Fuel gauge | MAX17048: VDD and CELL at the BAT node, CTG and QSTRT to GND, ALRT open | DS |
| Gauge decoupling | 0.1 µF at VDD | Confirm in step 4 |

The reversed-cell check: with the cell backwards, the only path from cell terminal to cell terminal runs through Q_REV. Its gate is negative and its body diode is reverse biased, so no current flows (Calc, by inspection). Test this with a current-limited supply first.

**DW01A vendor spread:** the thresholds differ between makers. The cited version has 4.3 V overcharge and 2.9 V overdischarge; others use about 2.4 V. Record the bought part. A protection threshold above the 3.0 V cutoff is not expected, but the 2.9 V variant leaves little margin.

### Starting after a cell swap (P11)

The datasheet says what wakes the part from a low-power state but does not state its state at first power-up.

| Situation | Expected | Evidence |
|---|---|---|
| Fresh cell above the recovery voltage | Normal state, discharge enabled | Not stated directly. Inferred from the DS recovery rule |
| Cell below VOD, part in low-power mode | Needs the charger to wake | DS |

No protection module is available for a test before ordering, so this is checked on the first board (A13). The open solder jumper is the fallback: closing it removes the DW01A from the path, leaving the charger limits and the 3.0 V cutoff, and P5 overdischarge backup and short-circuit protection are then lost. That would be a deliberate change for review.

## 03_Regulator_3V3

| Item | Value | Basis |
|---|---|---|
| R1, R2 (FB) | 511 kΩ, 91 kΩ, 1 % | 3.31 V (Calc); DS table values |
| Inductor | 0.47 µH, 4 × 4 mm, WPN4020HR47MT (alt. FXL0420-R47-M) | 13.4 A and 9.5 A listed, above the 5.75 A limit |
| C_IN | 22 µF, 10 V, 0805 × 2 | Effective at least 4 µF (DS) |
| C_OUT | 22 µF, 10 V, 0805 × 2 | Effective at least 7 µF (DS) |
| Switch | SPDT, common to the EN divider, one throw to VSYS | No load current through it |
| EN divider | 200 kΩ top, 100 kΩ bottom, 1 % | Ratio 1/3, cutoff 3.0 V (Calc) |
| C_EN | 100 nF | Time constant about 6.7 ms, ignores short dips (Calc) |
| MODE | 0 Ω to GND (PFM), unpopulated 0 Ω to VIN (PWM) | Lets us compare noise at bring-up |
| PG | Not used | Open |

The EN pin is not a charger or USB signal, so charging is independent of the switch (P8).

Off-state drain:

| Part | Typ, µA | Max, µA | Evidence |
|---|---|---|---|
| MAX17048 active | 23 | 40 | DS; hibernate about 4 µA |
| DW01A | 2 | 6 | DS |
| BQ24074 BAT | 4.3 | 6.5 | DS, at 85 °C |
| TPS63802 | 0.05 | 0.6 | DS |
| **Total** | **29** | **53** | Calc |

At 29 µA, a 2000 mAh cell lasts about 8 years with the system off (Calc, ignoring self-discharge).

## Cell path drop

| Element | mΩ | Evidence |
|---|---|---|
| Cell | 60 | Est |
| Holder contacts | 20 | Est |
| Q_REV | 40 | Est, confirm in step 4 |
| FS8205A | 60 | Est, confirm in step 4 |
| BQ24074 BAT to OUT | 75 | DS, 50–100 mV at 1 A |
| **Total** | **255** | **0.31 V at 1.22 A** (Calc) |

This meets the A6 limit of 0.35 V, raised from 0.3 V. The case is the full 1 A load at the cutoff, which is the sum of every peak at once. The cost is that the device stops slightly earlier under a heavy peak.

## 04_MCU

| Item | Value | Basis |
|---|---|---|
| Module | ESP32-C5-WROOM-1U-N8R8-V1.2 | Marauder target: 8 MB flash, PSRAM. The non-V1.2 listing is marked discontinued |
| Decoupling | 22 µF + 0.1 µF at pin 2 | DS peripheral schematic |
| EN | 10 kΩ pull-up, 1 µF | DS |
| GPIO27, GPIO28 | 10 kΩ pull-ups | Boot straps |
| Buttons | RESET to EN, BOOT to GPIO28 | |
| LED | XL-1615RGBC-WS2812B, 0.1 µF | LED datasheet 2.5–5.5 V; distributors list higher. Check at bring-up |
| Buzzer | MLT-8530, MMBT3904, 1 kΩ base, flyback diode | GPIO6 |
| VBAT divider | 200 kΩ / 100 kΩ, unpopulated | For JanOS |

## 05_USB_UART

| Item | Value | Basis |
|---|---|---|
| Bridge | CH343P: VDD5 and VBUS pins from USB VBUS, V3 on its own 100 nF, VIO from 3V3 | Dual supply recommended by the datasheet. No draw from the cell, and VIO does not back-feed VDD5 or V3 (DS) |
| USB lines | Direct to the connector, no series resistors | DS |
| DTR pin | No pull-down | A 3–8 kΩ pull-down at reset would select half-duplex mode (DS) |
| Auto-bootloader | Two MMBT3904 and two 10 kΩ, cross-coupled RTS/DTR to EN and GPIO28 | Standard ESP circuit |
| Fallback | CH340C (SOP-16), works at 3.3 V with V3 to VCC (DS) | Not pin compatible |

The CH343 datasheet V1.6 confirms the pinout and a built-in CDC driver path that needs no install. macOS behavior is checked at bring-up.

## 06_Display

| Item | Value |
|---|---|
| Socket | 1×14, 2.54 mm, 8.5 mm high |
| VCC | 3V3 |
| I2C pull-ups | Pads unpopulated; 10 kΩ on the display |

## 07_GPS

The GPS parts are optional. With them unpopulated, the board works and the external header stays usable.

| Item | Value | Basis |
|---|---|---|
| Module | ATGM336H-5N31 | 2.7–3.6 V, under 25 mA continuous, 100 mA peak (DS) |
| Supply filter | 10 µH, 0805, 10 µF | Corner 16 kHz; about 7700× at 1.4 MHz (Calc). Ferrites are weak at 1.4 MHz |
| Filter drop | About 0.08 V at 100 mA with 0.8 Ω | Est |
| Antenna feed | 47 nH from VCC_RF to the RF line | DS active antenna circuit |
| Antenna | U.FL, **active** ceramic antenna | Passive needs an extra LNA stage (DS) |
| RF trace | 50 Ω, short | DS |
| VBAT (backup) | Tied to the filtered VCC | No hot start; cold start each power-up |
| ON/OFF, nRESET | Confirm in step 4 | nRESET may float (DS) |
| External header | 1×5: VCC, GND, TXD, RXD, PPS, unpopulated | F9 |

Do not connect the on-board module and an external one at the same time, because both would drive GPIO14.

The antenna listed in the case BOM is unverified. Buy an antenna described as active, 3.3 V, with a U.FL lead.

## Part list

Stock and price are **not checked**: LCSC and JLCPCB pages did not return them here. Buy parts from your usual shops; the codes identify the part and the EasyEDA footprint.

Avoid AliExpress for the ESP32-C5 module, MAX17048, and TPS63802 if another source exists, because fake or reclaimed parts would be hard to diagnose.

| Sheet | Part | LCSC | Qty | Package |
|---|---|---|---|---|
| 01 | USB-C TYPE-C-31-M-12 | C165948 | 1 | 16-pin SMD |
| 01 | USBLC6-2SC6 | C323793 | 1 | SOT-23-6 |
| 01 | BQ24074RGTR | C54313 | 1 | QFN-16 3×3 |
| 02 | Battery holder, 18650, 1042 size class | C5249310 | 1 | SMT tabs |
| 02 | AO3400A | C20917 | 1 | SOT-23 |
| 02 | DW01A | C85680 | 1 | SOT-23-6 |
| 02 | FS8205A | C16052 | 1 | TSSOP-8 |
| 02 | MAX17048G+T10 | C2682616 | 1 | TDFN-8 2×2 |
| 03 | TPS63802DLAR | C704850 | 1 | VSON-10 3×2 |
| 03 | WPN4020HR47MT | C370628 | 1 | 4×4 mm |
| 03 | Slide switch MSK12C02 type | C47020938 | 1 | SMD, side |
| 04 | ESP32-C5-WROOM-1U-N8R8-V1.2 | C54951858 | 1 | Module |
| 04 | XL-1615RGBC-WS2812B | C5349954 | 1 | 1.6×1.5 mm |
| 04 | MLT-8530 | C94599 | 1 | 8.5×8.5 mm |
| 04 | MMBT3904 | C20526 | 1 | SOT-23 |
| 04 | Button TS-1187A-B-A-B | C318884 | 2 | 5.1×5.1 mm |
| 05 | CH343P | C2846043 | 1 | QFN-16 3×3 |
| 05 | MMBT3904 | C20526 | 2 | SOT-23 |
| 06 | Socket PM254-1-14-Z-8.5 | C2897377 | 1 | THT |
| 07 | ATGM336H-5N31 (optional) | C90770 | 1 | 9.7×10.1 mm |
| 07 | U.FL-R-SMT-1(01) (optional) | C598199 | 1 | SMD |

Plus 0603 resistors and capacitors, 0805 22 µF capacitors, and a 10 µH 0805 inductor. Step 4 lists exact values.

## Step 4 results

The schematic is in EasyEDA project `Marauder C5 Pancake`, seven sheets, as in `requirements-and-plan.md`, section 8.1.

| Check | Result | Evidence |
|---|---|---|
| Netlist against the design table | 79 parts, 65 nets, 0 differences; no single-pin nets | Exported netlist compared by script |
| Unused pins | Marked no-connect | Schematic |
| ERC | DW_VDD duplicate name fixed. Two warnings accepted: part names were set to values such as 10k, which differs from the library Name. Supplier Part codes are correct in the netlist | EasyEDA log, read by the reviewer; rerun pending |
| 3D models | 41 of 42 distinct parts; TPS63802 has none, use a 3 × 2 × 1 mm envelope | Library record |

Symbol pins checked against the datasheet: BQ24074, TPS63802, MAX17048, ESP32-C5-WROOM-1U, CH343P, ATGM336H.

Checked against typical use only, not against a datasheet: USBLC6-2SC6, XL-1615 LED, SW1, MLT-8530, TS-1187A, FS8205A, DW01A pin order.

Library part changes from step 3: 10 kΩ resistors use C98220, because the C25804 symbol has an oversized designator. Several resistor codes from step 3 pointed at the wrong value, and the list here uses the verified ones.

Test points are planned as pads on the PCB, not as schematic parts. J3 (debug) and J4 (spare) carry UART0, EN, BOOT, 3V3, VSYS, and the spare GPIO on unpopulated headers.
