# Power design (step 2)

Status: ready for review. This is the architecture and limits record from `power-pcb-design` step 2. Part values are sized in step 3.

Evidence labels:

- **DS**: datasheet value
- **Calc**: calculation in this file
- **Est**: estimate, to confirm
- **No data**: not available yet; not treated as a pass

## Scope of the analysis

The board has one switching converter, the DC-DC. The charger is linear, so it adds heat but no commutation loop. Gate drivers are inside the DC-DC, so there are no external gate loops.

All nets stay below 6 V. No safety standard applies, and insulation derivation is skipped; step 5 records this.

Parts: **BQ24074** charger and **TPS63802** buck-boost, confirmed in step 3. Values are in `circuit-design.md`.

## Power tree

```
USB-C 5 V ─ VBUS ─ BQ24074 IN        OUT = VSYS ─┬─ TPS63802 VIN ─ VOUT = 3V3 ─ loads
                         BAT ─┐                  └─ slide switch ─ divider ─ TPS63802 EN
                              │
Cell+ ─ BAT node ─ MAX17048
Cell− ─ reverse-protection N-FET ─ B− ─ protection FETs (DW01A + FS8205A) ─ GND
```

## Operating modes

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

## Cutoff with one switch

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

## Load cases

3V3 design current is 1.0 A from the budget in `requirements-and-plan.md`, section 6. Efficiency of 90 % is the datasheet's suggested assumption, not a value from its curves.

| Case | VIN to DC-DC, V | DC-DC mode | Input current at 1 A, A | Evidence |
|---|---|---|---|---|
| Empty cell | 3.0 | Boost, D = 0.09 | 1.22 | Calc |
| Nominal | 3.6 | Buck or buck-boost | 1.02 | Calc |
| Full cell | 4.2 | Buck | 0.87 | Calc |
| USB | 4.4 | Buck | 0.83 | Calc |

Inductor peak in boost at 3.0 V, with L = 0.47 µH and f = 2.1 MHz: **1.36 A**, or 1.63 A with the datasheet's 20 % margin (Calc). The datasheet gives f at VIN = 2.3 V, no load, so this frequency is approximate. Device current limit reaches 5.75 A max in boost (DS), so step 3 weighs the inductor rating against it. The datasheet's own inductors are rated 5.4 A and 5.5 A.

## Loop map

| Loop | Active when | Ordered path | Closing element | Return | Main uncertainty |
|---|---|---|---|---|---|
| L1: buck leg | VIN above about 3.3 V; both legs near 3.3 V | C_IN+ → VIN pin → internal high-side switch → L1 node ↔ internal low-side switch → GND pin → C_IN− | C_IN at VIN and GND pins | Inner ground plane directly under it | Switch edge time not in the datasheet |
| L2: boost leg | VIN below about 3.3 V; both legs near 3.3 V | C_OUT+ → VOUT pin → internal high-side switch → L2 node ↔ internal low-side switch → GND pin → C_OUT− | C_OUT at VOUT and GND pins | Inner ground plane directly under it | Same |
| L3: inductor | Always | L1 pin → inductor → L2 pin | — | None, this is a current path | Radiated field near GPS |
| L4: cell path | Battery powered | Cell+ → reverse FET → BAT/OUT → C_IN → DC-DC → GND → protection FETs → Cell− | C_IN and bulk capacitance | Ground plane and holder tab | Contact resistance of the holder |
| L5: TX peak | Wi-Fi TX | C_OUT or module bulk cap → 3V3 pin → module → GND | Module decoupling | Ground plane | Peak profile not known |

Transitions are inside the IC. The board controls only the capacitor position, the inductor position, and the ground return.

## Electrical net table

### Nodes

| Node | Ref. | Working | Peak | dv/dt | Constraint | Verification |
|---|---|---|---|---|---|---|
| VBUS / IN | GND | 5 V | 28 V rating, OVP at 10.2 to 10.8 V (DS, BQ24074) | Low | ESD at the connector; VBUS rating of the connector side parts | Step 8 review |
| VSYS (OUT) | GND | 3.0 to 4.5 V | 4.5 V (DS) | Low | Below 5.5 V TPS63802 input (DS) | Pass by design, Calc |
| BAT | GND | 3.0 to 4.23 V | 4.23 V (DS) | Low | — | Pass, DS |
| L1 | GND | 0 to VIN | 6 V DC, 9 V for under 10 ns (DS limit) | No data | Smallest copper, no vias if possible, away from GPS and FB | Step 6, step 8 |
| L2 | GND | 0 to VOUT | Same limit | No data | Same | Step 6, step 8 |
| 3V3 | GND | 3.3 V ±1 % FB | No data for load steps | Low | Within 3.0 to 3.6 V at the module (DS) | Measure |
| Cell− side of the FETs | GND | Up to about 0.1 V | Fault dependent | Low | Short, wide, away from sensitive refs | Step 8 |

### Branches

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

### Sensitive nets

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

## Charger heat

Power in the IC: P = (VIN − VOUT) × IIN + (VOUT − VBAT) × ICHG. Thermal resistance is 44.5 °C/W (DS, a JEDEC value, so the real board differs). Input is the 500 mA mode, with EN1 high and EN2 low.

| Case | ICHG, A | IIN, A | P, W | Rise, °C | Tj at 25 °C ambient |
|---|---|---|---|---|---|
| VBAT 3.0 V, device off | 0.45 | 0.475 | 0.92 | 41 | 66 °C |
| VBAT 3.6 V, device off | 0.45 | 0.475 | 0.65 | 29 | 54 °C |

Heat is no longer the limiting factor, including in a closed case.

## USB input current

**Decision:** Q9 option B, fixed USB500 mode: 450 to 500 mA from any port (DS). VIN-DPM is available in this mode, so weak ports are handled (DS).

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

## Acceptance criteria

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

## Open items

1. TI simulation model for the DC-DC: not obtained, no SPICE tool here.
2. Cell protection start-up (A13).
3. Holder contact resistance, by measurement.

## Sources

Checked 2026-11-05:

- [TPS63802 datasheet, SLVSEU9D](https://www.ti.com/lit/ds/symlink/tps63802.pdf)
- [BQ2407x datasheet, SLUS810N](https://www.ti.com/lit/ds/symlink/bq24074.pdf)
- [ESP32-C5-WROOM-1/1U datasheet v1.3](https://documentation.espressif.com/esp32-c5-wroom-1_wroom-1u_datasheet_en.pdf)
- [ATGM336H-5N manual](https://wiki.millerjs.org/_media/atgm336h-5n_datasheet.pdf)
