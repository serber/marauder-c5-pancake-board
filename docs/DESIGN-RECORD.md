# Marauder C5 Pancake Board — design record

Process: `power-pcb-design` skill, steps 1–9. Evidence labels: **DS** data sheet, **Calc** calculated, **Est** estimate, **No data**. Nothing has been simulated, built or measured. Requirements are in [SPEC.md](SPEC.md).

## Gate log

| Step | Date | Decision |
|---|---|---|
| 1 | 2026-11-05 | Approved. Target is the Pancake pinout with `pancake` firmware. No existing case: the board comes first. Display chosen, `VCC` from 3V3, no 5 V rail. Hand assembly on a hot table. Keystone 1042 class holder. Slide switch on DC-DC enable; power off cuts the whole system |
| 1a, 2 | 2026-11-06 | Approved. Fixed 500 mA USB input (was up to 1 A). Cell is swappable, on-board charging is an extra. One schematic sheet per function |
| 3 | 2026-11-06 | Approved. Reverse protection is a low-side N-FET. A6 limit raised to 0.35 V. DW01A start-up checked on the first board |
| 4 | 2026-11-06 | Approved. Seven sheets. CH343 core moved from 3V3 to VBUS per its data sheet. Module changed to the V1.2 listing. Name-attribute ERC warnings accepted |
| 5 | 2026-11-06 | Approved. The four-layer rule template could not be applied through the API, so a customized two-layer template is used |
| 1, 4–6 | 2026-11-06 | On-board GPS removed (U8, J6, L3, R29, RF_50R class); J2 kept. LED changed to SK6812MINI-HS on the display side. USB-C moved to the right edge. Header kept for the display. M3 holes, 3 mm corners |
| 6–9 | 2026-11-06 | Gates waived by the reviewer. Routing mode C chosen by the agent. No ordering |
| 9 | 2026-11-07 | Release exported again after the user's silkscreen moves. Only silkscreen layers changed; BOM and placement byte-identical |
| 9 | 2026-11-09 | Passives unified: all MLCC Samsung CL, all chip resistors Yageo RC. Values, footprints and ratings unchanged; 10 µF and 22 µF keep 10 V and 25 V. Only MPN and LCSC changed, DRC clean, Gerber unchanged |

## Tool inventory

| Tool | Where | Version |
|---|---|---|
| EasyEDA Pro | User's computer, driven through the `easyeda-api` bridge | 3.2.209 |
| Circuit simulator | — | None. No SPICE model for the DC-DC was obtained |

## Step 2: power design

- **DS**: datasheet value
- **Calc**: calculation in this file
- **Est**: estimate, to confirm
- **No data**: not available yet; not treated as a pass

### Scope of the analysis

The board has one switching converter, the DC-DC. The charger is linear, so it adds heat but no commutation loop. Gate drivers are inside the DC-DC, so there are no external gate loops.

All nets stay below 6 V. No safety standard applies, and insulation derivation is skipped; step 5 records this.

Parts: **BQ24074** charger and **TPS63802** buck-boost, confirmed in step 3. Values are in step 3.

### Power tree

```
USB-C 5 V ─ VBUS ─ BQ24074 IN        OUT = VSYS ─┬─ TPS63802 VIN ─ VOUT = 3V3 ─ loads
                         BAT ─┐                  └─ slide switch ─ divider ─ TPS63802 EN
                              │
Cell+ ─ BAT node ─ MAX17048
Cell− ─ reverse-protection N-FET ─ B− ─ protection FETs (DW01A + FS8205A) ─ GND
```

### Operating modes

| Mode | USB | Switch | VSYS | 3V3 | Charging | Notes |
|---|---|---|---|---|---|---|
| Off | No | Off | VBAT | Off | No | Drain: protection, gauge, DC-DC shutdown (45 nA typ, 600 nA max, DS) |
| Charge while off | Yes | Off | 4.4 V | Off | Yes | Bridge runs from VBUS and enumerates; the ESP stays off |
| Run on battery | No | On | VBAT − about 0.1 V | On | No | VBAT to OUT drop 50 mV typ, 100 mV max at 1 A (DS) |
| Run on USB | Yes | On | 4.3 to 4.5 V (DS) | On | Yes, reduced by load | Flash and CLI work here |
| No cell | Yes | On | 4.4 V, limited to 450–500 mA in | On | No | TX peaks can reset the device, because no cell backs up the 500 mA limit. Check at bring-up |
| Peak load on USB | Yes | On | Falls to VBAT − 40 mV | On | Paused | Battery supplement mode (DS) |
| Cell undervoltage | No | On | Below cutoff | Off | No | EN falling threshold, then cell protection |
| Protection tripped | — | — | — | Off | — | Cleared by the DW01A class recovery behavior, step 3 |
| Reversed cell | Any | Any | Must stay safe | — | No | Series reverse protection blocks it |

The bridge core runs from USB VBUS and only its I/O runs from 3V3, as its datasheet recommends. It draws nothing from the cell without USB, and VIO does not back-feed the core (P10). Flashing needs the switch on, because the ESP is off otherwise.

### Cutoff with one switch

The slide switch connects VSYS to a divider on the DC-DC EN pin. This covers P7 and P8 with no current through the switch beyond the divider.

| Item | Value | Evidence |
|---|---|---|
| EN threshold rising | 1.07 / 1.10 / 1.13 V | DS |
| EN threshold falling | 0.97 / 1.00 / 1.03 V | DS |
| Divider ratio for a 3.0 V falling cutoff | 1/3 | Calc |
| Falling cutoff | 2.91 / 3.00 / 3.09 V | Calc |
| Rising start | 3.21 / 3.30 / 3.39 V | Calc |
| EN at VSYS = 4.5 V | 1.5 V, below the 6 V limit | Calc |

**Risk:** near empty, a TX peak can drop VSYS to the cutoff, and the cell recovers above 3.3 V once the load is gone. The device can then restart repeatedly. Step 3 adds an RC on EN to ignore short dips. Check this at bring-up.

### Load cases

3V3 design current is 1.0 A from the budget in SPEC, Architecture. Efficiency of 90 % is the datasheet's suggested assumption, not a value from its curves.

| Case | VIN to DC-DC, V | DC-DC mode | Input current at 1 A, A | Evidence |
|---|---|---|---|---|
| Empty cell | 3.0 | Boost, D = 0.09 | 1.22 | Calc |
| Nominal | 3.6 | Buck or buck-boost | 1.02 | Calc |
| Full cell | 4.2 | Buck | 0.87 | Calc |
| USB | 4.4 | Buck | 0.83 | Calc |

Inductor peak in boost at 3.0 V, with L = 0.47 µH and f = 2.1 MHz: **1.36 A**, or 1.63 A with the datasheet's 20 % margin (Calc). The datasheet gives f at VIN = 2.3 V, no load, so this frequency is approximate. Device current limit reaches 5.75 A max in boost (DS), so step 3 weighs the inductor rating against it. The datasheet's own inductors are rated 5.4 A and 5.5 A.

### Loop map

| Loop | Active when | Ordered path | Closing element | Return | Main uncertainty |
|---|---|---|---|---|---|
| L1: buck leg | VIN above about 3.3 V; both legs near 3.3 V | C_IN+ → VIN pin → internal high-side switch → L1 node ↔ internal low-side switch → GND pin → C_IN− | C_IN at VIN and GND pins | Inner ground plane directly under it | Switch edge time not in the datasheet |
| L2: boost leg | VIN below about 3.3 V; both legs near 3.3 V | C_OUT+ → VOUT pin → internal high-side switch → L2 node ↔ internal low-side switch → GND pin → C_OUT− | C_OUT at VOUT and GND pins | Inner ground plane directly under it | Same |
| L3: inductor | Always | L1 pin → inductor → L2 pin | — | None, this is a current path | Radiated field near GPS |
| L4: cell path | Battery powered | Cell+ → reverse FET → BAT/OUT → C_IN → DC-DC → GND → protection FETs → Cell− | C_IN and bulk capacitance | Ground plane and holder tab | Contact resistance of the holder |
| L5: TX peak | Wi-Fi TX | C_OUT or module bulk cap → 3V3 pin → module → GND | Module decoupling | Ground plane | Peak profile not known |

Transitions are inside the IC. The board controls only the capacitor position, the inductor position, and the ground return.

### Electrical net table

#### Nodes

| Node | Ref. | Working | Peak | dv/dt | Constraint | Verification |
|---|---|---|---|---|---|---|
| VBUS / IN | GND | 5 V | 28 V rating, OVP at 10.2 to 10.8 V (DS, BQ24074) | Low | ESD at the connector; VBUS rating of the connector side parts | Step 8 review |
| VSYS (OUT) | GND | 3.0 to 4.5 V | 4.5 V (DS) | Low | Below 5.5 V TPS63802 input (DS) | Pass by design, Calc |
| BAT | GND | 3.0 to 4.23 V | 4.23 V (DS) | Low | — | Pass, DS |
| L1 | GND | 0 to VIN | 6 V DC, 9 V for under 10 ns (DS limit) | No data | Smallest copper, no vias if possible, away from GPS and FB | Step 6, step 8 |
| L2 | GND | 0 to VOUT | Same limit | No data | Same | Step 6, step 8 |
| 3V3 | GND | 3.3 V ±1 % FB | No data for load steps | Low | Within 3.0 to 3.6 V at the module (DS) | Measure |
| Cell− side of the FETs | GND | Up to about 0.1 V | Fault dependent | Low | Short, wide, away from sensitive refs | Step 8 |

#### Branches

Design currents add margin to the calculated worst case.

| Branch | RMS, A | Peak, A | Fault | Design current | Constraint | Verification |
|---|---|---|---|---|---|---|
| USB VBUS to IN | 0.50 max in USB500 mode (DS) | 0.50 | Charger limit | 1.0 A | Wide track | Step 8 copper check |
| Cell path, BAT to VIN | 1.22 at 1 A load (Calc) | Up to the DC-DC limit, 5.75 A (DS) | Ends at the protection overcurrent; value from step 3 | 2.0 A | Pour on two layers; several vias per layer change | Step 8 |
| DC-DC output to 3V3 | 1.0 | 2.0 (rating) | DC-DC current limit | 2.0 A | Pour | Step 8 |
| C_IN ripple | 0.43 at VIN 4.4 V (Calc) | No data | — | — | Smallest loop L1 | Step 6 estimate |
| C_OUT ripple | 0.32 in boost at 3.0 V (Calc) | No data | — | — | Smallest loop L2 | Step 6 estimate |
| Inductor | 1.22 | 1.36 at 1 A; 5.75 in limit | DC-DC current limit | Part rating | Short and wide, on the top layer | Step 8 |
| Charge current | 0.45 nominal, set in step 3 | 0.50 | Timer, thermal loop | 1.0 A | Wide track to the holder tab | Step 8 |

di/dt: No data, because the edge time is not in the datasheet. Loop inductance budgets stay open until step 3, which looks for a TI simulation model. Without one, the rule is the datasheet's layout guidance: capacitors as close as possible, with short, wide traces.

#### Sensitive nets

| Net | Function | Aggressors | Constraint | Verification |
|---|---|---|---|---|
| FB | DC-DC feedback, 500 mV | L1, L2 | Divider at the pin, returned to AGND; away from L1 and L2 (DS) | Step 8 |
| AGND | DC-DC control ground | GND pin currents | Joins power ground at one point close to the IC (DS) | Step 8 |
| EN divider | 1.0 to 1.1 V threshold | L1, L2 | Divider at the pin, RC at the pin | Step 8 |
| ISET, ILIM, TS | Charger programming | VBUS and charger heat | Resistors at the pins, returned to VSS | Step 8 |
| CELL at MAX17048 | Fuel gauge voltage | Cell path current | Kelvin from the holder tab, not from the shared pour | Step 8 |
| DW01A class CS | Overcurrent sense | Cell path current | Sensed at the FET, with a filter at the pin | Step 8 |
| GPS_3V3 at J2 | Below 50 mVpp ripple, for an ATGM336H class module (manual) | DC-DC ripple | Filter at J2 | Measure |
| USB D+/D− | USB full speed | SPI | Short, matched pair | Step 8 |

### Charger heat

Power in the IC: P = (VIN − VOUT) × IIN + (VOUT − VBAT) × ICHG. Thermal resistance is 44.5 °C/W (DS, a JEDEC value, so the real board differs). Input is the 500 mA mode, with EN1 high and EN2 low.

| Case | ICHG, A | IIN, A | P, W | Rise, °C | Tj at 25 °C ambient |
|---|---|---|---|---|---|
| VBAT 3.0 V, device off | 0.45 | 0.475 | 0.92 | 41 | 66 °C |
| VBAT 3.6 V, device off | 0.45 | 0.475 | 0.65 | 29 | 54 °C |

Heat is no longer the limiting factor, including in a closed case.

### USB input current

**Decision:** fixed USB500 mode: 450 to 500 mA from any port (DS). VIN-DPM is available in this mode, so weak ports are handled (DS).

Consequences:

| Item | Effect | Evidence |
|---|---|---|
| Charge current | Set to about 0.45 A, so ICHG stays below the input limit | Decision, R_ISET in step 3 |
| Charge time, device off, 2000 mAh | About 4.4 h, plus the taper at the end | Calc |
| Charge time, device off, 3200 mAh | About 7.1 h, plus the taper at the end | Calc |
| Device on while charging | Load takes from the same 500 mA, so charging is slower. Below 4.3 V on OUT, charge current drops | DS behavior |
| Peaks above 500 mA on USB | Battery supplement mode covers them | DS |
| No cell, USB only | No backup for peaks; resets are possible | Bring-up |
| Safety timer | Must allow a full charge of a 3500 mAh cell at 0.45 A; R_TMR set in step 3 | Step 3 |

### Acceptance criteria

| ID | Criterion | Limit | Source | Check | State |
|---|---|---|---|---|---|
| A1 | VSYS stays below DC-DC input maximum | ≤ 5.5 V | DS | Calc | Pass, 4.5 V max |
| A2 | Current of DC-DC at 3.0 V | ≥ 1 A | DS, 2 A at 2.3 V or above | Bring-up | Pass by DS |
| A3 | 3V3 at the module during TX | 3.0 to 3.6 V | DS | Measure | No data |
| A4 | Current into BQ24074 | ≤ 0.5 A from USB | DS | Calc | Pass, USB500 mode |
| A5 | Charger junction | ≤ 125 °C, regulated | DS | Measure | Calc pass, 66 °C at 25 °C ambient |
| A6 | Cell path drop at 1.22 A | ≤ 0.35 V | Decision | Calc, then measure | Pass: 0.31 V, Est |
| A7 | Falling cutoff at or above 2.9 V | 2.91 V min | Calc | Measure | Pass by Calc |
| A8 | Off drain | ≤ 50 µA | Decision | Measure | Pass at 29 µA typ; 53 µA worst case exceeds, Calc |
| A9 | J2 supply ripple | ≤ 50 mVpp | Manual | Measure | No data; filter gives about 7700× at 1.4 MHz, Calc |
| A10 | No restart loop near empty | Stops cleanly | Decision | Bring-up | No data |
| A11 | Reversed cell | No damage | P6 | Bring-up, current limited supply first | No data |
| A12 | Copper rise at design current | Set in step 5 | Step 5 | Calc | No data |
| A13 | Starts after inserting a charged cell, no USB | 10 of 10 tries | P11 | First-board bring-up | No data |

### Open items

1. TI simulation model for the DC-DC: not obtained, no SPICE tool here.
2. Cell protection start-up (A13).
3. Holder contact resistance, by measurement.

## Step 3: circuit

### Schematic sheets

One sheet per function, at the user's request.

| Sheet | Contents |
|---|---|
| 01_Power_Input | USB-C, VBUS protection, charger |
| 02_Battery | Holder, cell protection, reverse protection, MAX17048 |
| 03_Regulator_3V3 | Switch, EN divider, TPS63802 stage, test points |
| 04_MCU | ESP32-C5 module, decoupling, EN, strapping, buttons, LED, buzzer |
| 05_USB_UART | USB-UART bridge, auto-bootloader |
| 06_Display | Display socket, I2C pull-up pads |
| 07_GPS | Supply filter, external GPS header |

### Simulation

No simulation ran. No SPICE tool is installed in this environment, and no DC-DC model has been obtained. The loop-inductance budgets in step 2 therefore stay open. The layout follows the datasheet guidance instead, and bench measurement replaces simulation in Check on the first board.

### 01_Power_Input

| Item | Value | Basis |
|---|---|---|
| Input mode | USB500: EN1 to OUT, EN2 to GND | 450–500 mA (DS) |
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

### 02_Battery

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

#### Starting after a cell swap (P11)

The datasheet says what wakes the part from a low-power state but does not state its state at first power-up.

| Situation | Expected | Evidence |
|---|---|---|
| Fresh cell above the recovery voltage | Normal state, discharge enabled | Not stated directly. Inferred from the DS recovery rule |
| Cell below VOD, part in low-power mode | Needs the charger to wake | DS |

No protection module is available for a test before ordering, so this is checked on the first board (A13). The open solder jumper is the fallback: closing it removes the DW01A from the path, leaving the charger limits and the 3.0 V cutoff, and P5 overdischarge backup and short-circuit protection are then lost. That would be a deliberate change for review.

### 03_Regulator_3V3

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

### Cell path drop

| Element | mΩ | Evidence |
|---|---|---|
| Cell | 60 | Est |
| Holder contacts | 20 | Est |
| Q_REV | 40 | Est, confirm in step 4 |
| FS8205A | 60 | Est, confirm in step 4 |
| BQ24074 BAT to OUT | 75 | DS, 50–100 mV at 1 A |
| **Total** | **255** | **0.31 V at 1.22 A** (Calc) |

This meets the A6 limit of 0.35 V, raised from 0.3 V. The case is the full 1 A load at the cutoff, which is the sum of every peak at once. The cost is that the device stops slightly earlier under a heavy peak.

### 04_MCU

| Item | Value | Basis |
|---|---|---|
| Module | ESP32-C5-WROOM-1U-N8R8-V1.2 | Marauder target: 8 MB flash, PSRAM. The non-V1.2 listing is marked discontinued |
| Decoupling | 22 µF + 0.1 µF at pin 2 | DS peripheral schematic |
| EN | 10 kΩ pull-up, 1 µF | DS |
| GPIO27, GPIO28 | 10 kΩ pull-ups | Boot straps |
| Buttons | RESET to EN, BOOT to GPIO28 | |
| LED | SK6812MINI-HS, VDD from DCDC_SW, 0.1 µF | Rated 3.7–5.5 V; the rail runs 3.0–4.4 V, so it can fail on a low cell. DIN high needs 0.65–0.7 × VDD, and 3.3 V meets it |
| Buzzer | MLT-8530, MMBT3904, 1 kΩ base, flyback diode | GPIO6 |
| VBAT divider | 200 kΩ / 100 kΩ, unpopulated | For JanOS |

### 05_USB_UART

| Item | Value | Basis |
|---|---|---|
| Bridge | CH343P: VDD5 and VBUS pins from USB VBUS, V3 on its own 100 nF, VIO from 3V3 | Dual supply recommended by the datasheet. No draw from the cell, and VIO does not back-feed VDD5 or V3 (DS) |
| USB lines | Direct to the connector, no series resistors | DS |
| DTR pin | No pull-down | A 3–8 kΩ pull-down at reset would select half-duplex mode (DS) |
| Auto-bootloader | Two MMBT3904 and two 10 kΩ, cross-coupled RTS/DTR to EN and GPIO28 | Standard ESP circuit |
| Fallback | CH340C (SOP-16), works at 3.3 V with V3 to VCC (DS) | Not pin compatible |

The CH343 datasheet V1.6 confirms the pinout and a built-in CDC driver path that needs no install. macOS behavior is checked at bring-up.

### 06_Display

| Item | Value |
|---|---|
| Socket | 1×14, 2.54 mm, 8.5 mm high |
| VCC | 3V3 |
| I2C pull-ups | Pads unpopulated; 10 kΩ on the display |

### 07_GPS

The on-board GPS was removed. J2 connects an external module.

| Item | Value | Basis |
|---|---|---|
| J2 | 1×5, 2.54 mm: GPS_3V3, GND, GPS_TXD, GPS_RXD, GPS_PPS | F9 |
| Supply filter | 10 µH, 0805, 10 µF, 100 nF | Corner 16 kHz; about 7700× at 1.4 MHz (Calc). Ferrites are weak at 1.4 MHz |
| Filter drop | About 0.08 V at 100 mA with 0.8 Ω | Est |
| PPS | Not connected to the ESP | No firmware uses it |

GPS_TXD (module TX) goes to GPIO14, and GPS_RXD goes to GPIO13. With no module fitted, GPIO14 has nothing driving it. The firmware should show no GPS. Check this at bring-up.

## Step 4: schematic audit

The schematic is in EasyEDA project `Marauder C5 Pancake`, seven sheets, as in the schematic sheet list in step 3.

| Check | Result | Evidence |
|---|---|---|
| Netlist against the design table | 75 parts, 62 nets, 0 differences; one single-pin net, GPS_PPS at J2 | Exported netlist compared by script |
| Unused pins | Marked no-connect | Schematic |
| ERC | DW_VDD duplicate name fixed. Two warnings accepted: part names were set to values such as 10k, which differs from the library Name. Supplier Part codes are correct in the netlist | EasyEDA log, read by the reviewer; rerun pending |
| 3D models | 38 of 39 distinct parts; TPS63802 has none, use a 3 × 2 × 1 mm envelope | Library record |

Symbol pins checked against the datasheet: BQ24074, TPS63802, MAX17048, ESP32-C5-WROOM-1U, CH343P.

Checked against typical use only, not against a datasheet: USBLC6-2SC6, XL-1615 LED, SW1, MLT-8530, TS-1187A, FS8205A, DW01A pin order.

Library part changes from step 3: 10 kΩ resistors use C98220, because the C25804 symbol has an oversized designator. Several resistor codes from step 3 pointed at the wrong value, and the list here uses the verified ones.

Test points are planned as pads on the PCB, not as schematic parts. J3 (debug) and J4 (spare) carry UART0, EN, BOOT, 3V3, VSYS, and the spare GPIO on unpopulated headers.

## Step 5: rules and stack-up

### Stack-up

JLC04161H-7628, 1.6 mm, four layers. Thicknesses come from the [generated stack-up file](https://github.com/gsuberland/jlcpcb_autogenerated_stackups) for this template. Check them against the JLCPCB order page.

| Layer | Thickness, mm | Use |
|---|---|---|
| L1 Top | 0.035 (1 oz) | All parts, signals, cell path, DC-DC |
| Prepreg 7628 | 0.2104 | |
| L2 Inner1 | 0.0152 (0.5 oz) | Solid GND, no routing |
| Core | 1.065 | |
| L3 Inner2 | 0.0152 (0.5 oz) | 3V3 pour, plus short signal escapes |
| Prepreg 7628 | 0.2104 | |
| L4 Bottom | 0.035 (1 oz) | GND pour, signals as needed |

Four copper layers are set in EasyEDA. Assigning L2 and L3 as planes happens with the ground plan in step 7a.

### Voltage classes

All nets stay below 6 V in operation. VBUS tolerates up to 28 V at the charger, and its OVP trips at 10.2–10.8 V (DS). No safety standard applies, so no insulation derivation was done. Spacing follows the fabricator rules. This is not a statement of safety compliance.

### Clearance and fabrication

| Rule | Value | Source | Margin |
|---|---|---|---|
| Track minimum | 0.127 mm | Decision | JLCPCB 0.09 mm for 4–6 layers |
| Track default | 0.2 mm | Decision | |
| Track to track | 0.102 mm | EasyEDA JLCPCB template | Plan routes at 0.127 mm or more |
| Via | 0.3 mm drill, 0.6 mm pad | Template default | JLCPCB 0.2 / 0.45 mm |
| Pour clearance | 0.254 mm | Template | |
| Board outline clearance | 0.3 mm | Template | |

EasyEDA stays on a custom copy of its JLCPCB template. The API rejected the four-layer template, so these values come from the two-layer one with changes. Its via and track limits are stricter than the four-layer capability, which is safe.

### Conductor capability

IPC-2221 estimate at 10 °C rise (Calc): I = k × ΔT^0.44 × A^0.725, k = 0.048 outer, 0.024 inner. This is a formula estimate, not a thermal analysis.

| Current, A | Outer 1 oz, mm | Inner 0.5 oz, mm |
|---|---|---|
| 0.5 | 0.12 | 0.60 |
| 1.0 | 0.30 | 1.56 |
| 1.5 | 0.53 | 2.73 |
| 2.0 | 0.78 | 4.07 |

Inner layers need far more width, so high-current paths run on L1 and L4, and L3 carries 3V3 as a wide pour.

| Net class | Nets | Design current | Rule | EasyEDA |
|---|---|---|---|---|
| POWER_2A | BAT, VSYS, 3V3, CELL_N, B_N, FS_D | 2.0 A | Default 1.0 mm or a pour; rule minimum 0.2 mm for low-current branches | Track rule on class |
| VBUS_1A | VBUS | 1.0 A | Default 0.5 mm; rule minimum 0.2 mm for branches | Track rule on class |
| SWITCH | DCDC_L1, DCDC_L2 | Part rating | Short and wide on L1, no vias | Class, placement note |
| SENSE | DCDC_FB, DCDC_EN, CHG_ISET, CHG_ILIM, CHG_TMR, CHG_TS | Signal | Default width, away from L1/L2 | Class, routing note |
| SPI | SPI_SCK, SPI_MOSI, SPI_MISO | Signal | Default width | Class |
| USB | USB_DP, USB_DN | Signal | Short, no pair rule | See step 7 |

**Vias:** 0.3 mm drill with 18 µm plating is treated as an inner conductor of its barrel width, about 0.7 A (Calc). Use at least 3 vias at each POWER_2A layer change. Prefer staying on L1.

### Holder footprint

The library footprint BAT-SMD_1042P has pad centers 78.7 mm apart, with 8.0 × 7.0 mm pads, so the span is 86.7 mm. It fits the 98 mm board and clears H1, H3, and the socket (Calc). Pin 2 is BAT (+) and pin 1 is CELL_N (−).

Measure the bought holder against this before ordering.

### Traceability

| Constraint from step 2 | Implementation | Check |
|---|---|---|
| Cell path design current | POWER_2A rule, via count | DRC, step 8 |
| C_IN and C_OUT at the DC-DC pins | Placement note | Step 6 image |
| L1/L2 smallest copper | SWITCH class, placement note | Step 6, step 8 |
| FB, EN away from L1/L2 | SENSE class, routing note | Step 8 |
| AGND joined near the IC | Routing note. Schematic uses one GND net | Step 8 |
| CELL Kelvin at the holder tab | Routing note | Step 8 |
| DW01A sense at the FET | Routing note | Step 8 |
| GPS filter at J2 | Placement note | Step 6 |
| Copper rise (A12) | Conductor table above | Step 8 |

### Notes

Schematic and netlist are imported into PCB1: 79 parts, 65 nets. Net-class rules were read back from EasyEDA after saving.

## Step 6: placement

### Board

| Item | Value |
|---|---|
| Outline | 55.50 × 98.00 mm, 3.0 mm corner radius |
| Holes | 4 × Ø3.2 mm for M3, plated, Ø5.0 mm pad on GND, at (3, 3), (52.5, 3), (3, 95), (52.5, 95) |

The corner radius is concentric with the holes.

### Block floorplan

| Block | Area, mm | Parts | Reason |
|---|---|---|---|
| Battery | X 0–21, Y 6–92 | BT1 | + tab at the top, − tab at the bottom |
| USB-C | X 49–56, Y 79–88 | J1, U1, R1, R2 | Right edge, beside SW1 |
| Charger | X 23–33, Y 79–91 | U2, charger passives | Beside the + tab |
| Status LEDs | X 23–30, Y 92–96 | D1, D2, R7, R8 | Top edge |
| Fuel gauge | X 15–22, Y 93–98 | U4, C5 | At the + tab for a Kelvin connection |
| USB-UART | X 34–50, Y 86–96 | U7, Q4, Q5 | Beside USB-C |
| DC-DC | X 29–46, Y 62–78 | U5, L1, C6–C10, R13–R18 | Between the cell path and the module |
| Power switch | X 50–56, Y 68–76 | SW1 | On the right edge, lever outward |
| MCU | X 31–55, Y 23–45 | U6, C11–C13, R19–R21 | Close to the socket; Y 47–60 stays free for routing |
| Indicators | X 22–31, Y 31–59 | BZ1, LED1, Q3, D3 | Beside the module |
| Debug headers | X 21–34, Y 22–29 | J3, J4 | Unpopulated |
| External GPS | X 39–52, Y 14–21 | J2, L2, C18, C19 | Farthest from the DC-DC inductor |
| Cell protection | X 22–37, Y 11–23 | Q1, Q2, U3, R9–R11, C4 | At the − tab |
| Buttons | X 34–48, Y 6–14 | SW2, SW3 | Bottom edge |
| Display socket | Y 2, display side | J5 | Fixed by the display |
| RGB LED | (45.5, 94.4), display side | LED1, C14 | Side window in the case |

### Socket check

J5 sits on the bottom layer. Read back from EasyEDA: pin 1 is at X = 44.26 and pin 14 is at X = 11.24, Y = 2.00, which matches SPEC, Mechanics.

### DC-DC layout

| Item | Position |
|---|---|
| U5 | (38.0, 71.0) |
| C7, C8 (input) | VSYS pad 1.6 mm from pin 10 |
| C9, C10 (output) | 3V3 pad 1.6 mm from pin 6 |
| L1 | Right of U5, pad 1 above pin L1 and pad 2 below pin L2, so the nets do not cross |

The capacitor ground pads reach the IC ground through L2. This is an estimate of a compact loop, and it has no inductance budget, because no simulation ran.

### Parts placed outside their group

| Part | Why |
|---|---|
| U4, C5 | At the + tab, far from the battery block's − end, for sensing |
| R10, C4 | Near U3, which is at the − tab, so VDD runs from the + tab along the board as a low-current trace |
| C11, C12 | At module pin 2 |

### Spacing pass for silkscreen

The reviewer asked for more room for the designators on 2026-11-06. Blocks and power copper stayed in place; only tight columns of passives moved.

| Parts | Change |
|---|---|
| C11, C12, R19, C13, R20, R21 | Column at X 31.8, pitch 2.0 → 2.7 mm, Y 44.0 to 30.5 |
| R3, R4, R5, R6 | Y 80.5 → 81.0, pitch 2.0 → 2.2 mm, X 24.5 to 31.1 |
| C6, R15 | Y 72.5 and 69.9, pitch 2.6 mm above R16 |
| R18, R17 | Y 64.7 and 62.1, pitch 2.6 mm below R16 |

Designators went from 1.14 mm to 1.0 mm and were placed by script next to their parts, clear of pads, holes, bodies, and each other. Nothing is hidden under the holder or the module.

The reviewer then adjusted the designators by hand in EasyEDA. SW2 and SW3 show `RESET` and `BOOT` labels instead of their designators, 1.14 mm high, above the buttons.

C11 now sits about 3.7 mm from module pin 2. C12 stays about 1 mm from it, and both connect to the L3 3V3 pour.

### Routability

| Metric | Value | Evidence |
|---|---|---|
| Ratsnest length, GND excluded | About 1830 mm before routing | Script, minimum spanning tree per net |
| Ratsnest crossings | About 340 before routing | Script, an upper estimate |
| Most congested | X 30–40, Y 10–40 | Script, 10 mm grid |
| Overlaps | None, by part bounding boxes | Script |
| Beyond the board edge | J1 by 1.5 mm, SW1 by 1.0 mm, intended | Script |

The congested areas are where display and I2C lines leave the left module pins for the socket. Lines from the right module pins must go around the module or pass under it on L4, so the display nets take L4 for most of their length.

### Trade-offs left open

1. **Module rotation.** Turning U6 by 90° could shorten the display nets, but it takes the width of the electronics zone. Not evaluated.
2. **Buttons.** SW2 and SW3 are near the bottom edge on the rear. Top-pressed buttons need a push rod or an opening in the case.
3. **Height.** All parts sit on the rear, next to the holder, and are lower than the cell. Holder height is still unmeasured.

## Step 7: routing

Gates for steps 6–9 were waived by the reviewer on 2026-11-06.

Mode **C, hybrid**. Power and critical nets were routed by script. Signals were routed by the EasyEDA autorouter on L1 and L4, with those nets kept in place. Some nets were ripped up and rerouted when they conflicted with USB.

### Copper

| Layer | Content |
|---|---|
| L1 | Parts, signals, cell path, DC-DC, GND pour |
| L2 | Solid GND pour |
| L3 | 3V3 pour, plus USB_CC2 as a short track |
| L4 | Signals, VBUS, GND pour |

Pours are inset 0.3 mm from the board edge.

### Critical nets

| Net | Route |
|---|---|
| BAT | L1, 1.0 mm from the + tab to C3, 0.8 mm, then 0.45 mm into U2 |
| VSYS | L1, 0.4 mm for about 4 mm near U2, then 0.8 mm to C7, C8 |
| 3V3 | L1 0.8 mm at C9, C10, then 3 vias to the L3 pour |
| CELL_N | L1, 1.0 mm from the − tab to Q1 |
| B_N | L1, 0.6 mm from Q1 to Q2 |
| FS8205A source to GND | L1 0.6 mm, 3 vias to L2 and L4 |
| DCDC_L1, DCDC_L2 | L1, 0.25 mm neck at the IC pins, then 0.9 mm to L1 |
| AGND | Short stub and its own via to L2 |
| VBUS | L4, 0.5 mm from USB-C to the charger |
| USB D+/D− | J1 pins joined with vias; about 22–25 mm to U7, no impedance control, no matched pair |

### Reroute after the spacing pass

CHG_ILIM, CHG_ISET, CHG_TMR, CHG_TS, DCDC_EN, DCDC_FB, DCDC_MODE, ESP_BOOT, ESP_EN, RGB_DIN, and SPARE_IO0 were ripped up and rerouted, with new GND and 3V3 stubs at the moved parts. CHG_TS and CHG_ISET were routed by hand, because the autorouter failed on CHG_ISET. Critical nets in the table above did not change.

ESP_BOOT runs mainly on L4 at X 36.6, as before. ESP_EN now has a 25 mm run on L1 at X 36, Y 43 to 68. Neither net has an impedance or length limit.

### Exceptions

| Item | Reason | Effect |
|---|---|---|
| POWER_2A minimum set to 0.2 mm | Low-current branches, such as BAT to R9, R10, R23, U4 | Main paths checked by hand in step 8 |
| VSYS 0.4 mm near U2 | Autorouted CHG_ILIM, CHG_STAT, CHG_TMR nearby | About 1.2 A peak, short |
| USB pair not defined in the end | Stayed flagged as a pair error after hand routing; USB full speed is forgiving | Check enumeration at bring-up |
| USB_CC2 on L3 | No route on L1 or L4 | Small slit in the 3V3 pour |
| Vias in the module ground pad | Connect the pad sections to L2 | Tented under the module |

## Step 8: verification

EasyEDA and script checks only: no simulation, bench test or thermal analysis with the real copper.

### Check record

| Check | Result | Evidence |
|---|---|---|
| Schematic against the design table | 75 parts, 62 nets, 0 differences | Netlist compare |
| PCB against the design table | 75 of 75 parts, 0 pad-net differences | PCB pads compare |
| PCB DRC, strict | 0 errors, including connectivity | EasyEDA, after pour refill, 2026-11-07 |
| Dangling track ends | None open, 3 removed after the reroute. A 0.05 mm VSYS tail at U2 ends inside the copper of its junction | Script, 2026-11-07. DRC does not report these |
| Designators against pads, holes, and each other | 0 overlaps at 1.0 mm text; C12 and Q2 labels sit within 0.15 mm of a pad after manual moves. The silkscreen moves of 2026-11-07 put no stroke on a mask opening or within 0.15 mm of the board edge; label-to-label spacing was not rechecked | Script, text bounding boxes; Gerber compare on 2026-11-07 |
| Display socket | Pin 1 at X 44.26, pin 14 at X 11.24 | Read from EasyEDA |
| Part overlaps | None | Bounding boxes, script |
| Edge and holder margin | Parts about 1.5 mm or more away; J1 and SW1 at the edge by design | Bounding boxes, script |
| 3D fit with the display and cell | **Not done** | Open the STEP file |
| Schematic ERC, strict | 0 errors, 3 warnings | `sch_Drc.check` on 2026-11-06. The API returns only the count, so read the warning text in the UI |
| Copper near M3 hardware | Non-GND copper is at least 2.76 mm from any hole center, Q5_B at H4 | Script, on L1 and L4 |

### Copper capability

| Path | Current | Narrowest | Estimate |
|---|---|---|---|
| BAT, main | 1.22 A peak | 0.45 mm, short neck | 0.40 mm needed for about 1.2 A at 10 °C |
| VSYS near U2 | 1.22 A peak | 0.4 mm, about 4 mm | At the limit, short |
| CELL_N | 1.22 A | 1.0 mm | Pass |
| B_N | 1.22 A | 0.6 mm | Pass |
| 3V3 to L3 | 1 A | 3 vias | Pass, about 0.7 A per via |
| GND at FS8205A | 1.22 A | 3 vias | Pass |
| VBUS | 0.5 A | 0.5 mm | Pass |

Estimates use the IPC-2221 formula in step 5, at peak current as if continuous.

### Net-table constraints

| Constraint | Result |
|---|---|
| C_IN, C_OUT at the DC-DC pins | Met: 3V3 and VSYS pads about 1.6 mm from the pins |
| FB away from L1, L2 | Met by layout: FB on the far side of the IC |
| AGND joins near the IC | Met through L2, not a direct net tie |
| CELL Kelvin at the tab | **Not met as designed:** U4 takes BAT from the autorouted branch. It is low current, but check the voltage reading |
| DW01A sense at the FET | R11 sits between the sense pin and GND near Q2 |
| Loop-inductance budgets | **Unresolved**, no model and no simulation |

### Acceptance criteria now

| ID | State |
|---|---|
| A1, A2, A4, A6, A7 | Pass by calculation |
| A3, A5, A9, A10, A11, A12, A13 | Measure at bring-up |
| A8 | 29 µA typ, 53 µA worst case, plus the LED and switch are off the cell |

### Known risks for the first board

1. **LED below its rating.** SK6812MINI-HS is rated 3.7–5.5 V, and DCDC_SW runs 3.0–4.4 V. It can fail below about 3.7 V on the cell.
2. **DW01A start-up** after a cell swap is unverified. R12 is the fallback, and it removes short-circuit protection.
3. **Backlight** can be dim at VCC = 3.3 V.
4. **GPIO14** floats with no GPS on J2.
5. **Holder** tab spacing and height are not measured against the bought part.
6. **M3 hardware at H4.** A Ø5.5 mm head clears Q5_B by 0.01 mm, and a Ø7 mm washer sits over Q5_B. The tracks are under solder mask, so use an insulating washer or none there.
7. **Stack-up** must be set to JLC04161H-7628, 1.6 mm, 1 oz outer, 0.5 oz inner when ordering.

### Part availability

Read on 2026-11-06 from the LCSC product API. Stock changes daily, so check again before buying.

| Part | Refs | LCSC | Stock | Note |
|---|---|---|---|---|
| ESP32-C5-WROOM-1U-N8R8-V1.2 | U6 | C54951858 | **0** | Search snippets show Mouser 356-ESP32C5WRM1UN8R8 in stock; not confirmed on a live page. See the revision note below |
| BQ24074RGTR | U2 | C54313 | **0** | Search snippets show Mouser and DigiKey in stock; not confirmed on a live page |
| FS8205A | Q2 | C16052 | **0** | Commodity part, sold by many shops. Check the TSSOP-8 pinout of any substitute |
| CL10B104KB8NNNC | C4, C5, C6, C12, C14, C16, C17, C19 | C1591 | Not checked | Replaced `CC0603KRX7R9BB104` on 2026-11-09. Any 100 nF X7R 50 V 0603 fits |
| RC0603FR-07 and RC0603JR-07 series | R1, R2, R4–R8, R10, R11, R13–R15, R17, R18, R22 | see BOM | Not checked | Replaced UNI-ROYAL `0603WAF` parts on 2026-11-09 |
| 1042, WPN4020HR47MT, SDFL2012S100KTF, USBLC6-2SC6, DW01A, TPS63802DLAR | BT1, L1, L2, U1, U3, U5 | see links | **Unknown** | The API returned no record, which does not show the part is gone. Check by hand |
| Other BOM lines | | | 1,950 or more | Minimum order is above the quantity needed on many lines |

**ESP32-C5 revision.** Espressif PCN20251201 moves ESP32-C5-WROOM-1U-N8R8 from chip revision v1.0 to v1.2 under the same part name. A module sold as plain N8R8 can carry either revision, so ask the seller which one it is. The change fixes a PSRAM consistency issue and an SRAM corruption issue. LCSC C51950748 (plain N8R8) and C54951858 (N8R8-V1.2) use the same EasyEDA footprint record, and the PCN describes a chip metal-layer change with the same part number, so both fit U6. The module marking tells them apart: MCXXXX is v1.0, MDXXXX is v1.2. Do not set "Minimum Supported ESP32-C5 Revision" to v1.2 in firmware for a v1.0 module.

Sources: [PCN20251201](https://documentation.espressif.com/en/PCN20251201_Upgrade_of_ESP32-C5_Series_Products.html), [DigiKey listing](https://www.digikey.com.au/en/products/detail/espressif-systems/ESP32-C5-WROOM-1U-N8R8/27566801).

## Main risks

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

## Check on the first board

1. Without the ESP: power rails, quiescent current, switch. Insert a charged cell without USB 10 times (A13).
2. Protection: reversed cell, discharge and charge thresholds, short circuit.
3. Charging: with the device off and on.
4. USB: port detection, auto-bootloader, flashing `pancake`.
5. Display, touch, SD, battery percentage, LED, buzzer, button.
6. Wi-Fi 2.4/5 GHz, BLE, GPS through J2, and without GPS.
7. Heating and voltage droop during TX peaks at low charge.
8. Fit of the display and the cell holder; STEP check for the case.
