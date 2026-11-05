# Verification (step 8)

Status: EasyEDA and script checks only. **No simulation, no bench test, no thermal analysis with the real copper.** The board is not validated.

## Check record

| Check | Result | Evidence |
|---|---|---|
| Schematic against the design table | 75 parts, 62 nets, 0 differences | Netlist compare |
| PCB against the design table | 75 of 75 parts, 0 pad-net differences | PCB pads compare |
| PCB DRC, strict | 0 errors, including connectivity | EasyEDA, after pour refill |
| Display socket | Pin 1 at X 44.26, pin 14 at X 11.24 | Read from EasyEDA |
| Part overlaps | None | Bounding boxes, script |
| Edge and holder margin | Parts about 1.5 mm or more away; J1 and SW1 at the edge by design | Bounding boxes, script |
| 3D fit with the display and cell | **Not done** | Open the STEP file |

Schematic ERC after the last changes was not read in the EasyEDA UI.

## Copper capability

| Path | Current | Narrowest | Estimate |
|---|---|---|---|
| BAT, main | 1.22 A peak | 0.45 mm, short neck | 0.40 mm needed for about 1.2 A at 10 °C |
| VSYS near U2 | 1.22 A peak | 0.4 mm, about 4 mm | At the limit, short |
| CELL_N | 1.22 A | 1.0 mm | Pass |
| B_N | 1.22 A | 0.6 mm | Pass |
| 3V3 to L3 | 1 A | 3 vias | Pass, about 0.7 A per via |
| GND at FS8205A | 1.22 A | 3 vias | Pass |
| VBUS | 0.5 A | 0.5 mm | Pass |

Estimates use the IPC-2221 formula in `design-rules.md`, at peak current as if continuous.

## Net-table constraints

| Constraint | Result |
|---|---|
| C_IN, C_OUT at the DC-DC pins | Met: 3V3 and VSYS pads about 1.6 mm from the pins |
| FB away from L1, L2 | Met by layout: FB on the far side of the IC |
| AGND joins near the IC | Met through L2, not a direct net tie |
| CELL Kelvin at the tab | **Not met as designed:** U4 takes BAT from the autorouted branch. It is low current, but check the voltage reading |
| DW01A sense at the FET | R11 sits between the sense pin and GND near Q2 |
| Loop-inductance budgets | **Unresolved**, no model and no simulation |

## Acceptance criteria now

| ID | State |
|---|---|
| A1, A2, A4, A6, A7 | Pass by calculation |
| A3, A5, A9, A10, A11, A12, A13 | Measure at bring-up |
| A8 | 29 µA typ, 53 µA worst case, plus the LED and switch are off the cell |

## Known risks for the first board

1. **LED below its rating.** SK6812MINI-HS is rated 3.7–5.5 V, and DCDC_SW runs 3.0–4.4 V. It can fail below about 3.7 V on the cell.
2. **DW01A start-up** after a cell swap is unverified. R12 is the fallback, and it removes short-circuit protection.
3. **Backlight** can be dim at VCC = 3.3 V.
4. **GPIO14** floats with no GPS on J2.
5. **Holder** tab spacing and height are not measured against the bought part.
6. **Stack-up** must be set to JLC04161H-7628, 1.6 mm, 1 oz outer, 0.5 oz inner when ordering.
