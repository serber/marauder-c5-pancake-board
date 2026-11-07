# Verification (step 8)

Status: EasyEDA and script checks only. **No simulation, no bench test, no thermal analysis with the real copper.** The board is not validated.

## Check record

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
6. **M3 hardware at H4.** A Ø5.5 mm head clears Q5_B by 0.01 mm, and a Ø7 mm washer sits over Q5_B. The tracks are under solder mask, so use an insulating washer or none there.
7. **Stack-up** must be set to JLC04161H-7628, 1.6 mm, 1 oz outer, 0.5 oz inner when ordering.

## Part availability

Read on 2026-11-06 from the LCSC product API. Stock changes daily, so check again before buying.

| Part | Refs | LCSC | Stock | Note |
|---|---|---|---|---|
| ESP32-C5-WROOM-1U-N8R8-V1.2 | U6 | C54951858 | **0** | Search snippets show Mouser 356-ESP32C5WRM1UN8R8 in stock; not confirmed on a live page. See the revision note below |
| BQ24074RGTR | U2 | C54313 | **0** | Search snippets show Mouser and DigiKey in stock; not confirmed on a live page |
| FS8205A | Q2 | C16052 | **0** | Commodity part, sold by many shops. Check the TSSOP-8 pinout of any substitute |
| CC0603KRX7R9BB104 | C4, C5, C6, C12, C14, C16, C17, C19 | C14663 | 100 | Minimum order 50. Any 100 nF X7R 0603 fits |
| 1042, WPN4020HR47MT, SDFL2012S100KTF, USBLC6-2SC6, DW01A, TPS63802DLAR | BT1, L1, L2, U1, U3, U5 | see links | **Unknown** | The API returned no record, which does not show the part is gone. Check by hand |
| Other 31 BOM lines | | | 1,950 or more | Minimum order is above the quantity needed on many lines |

**ESP32-C5 revision.** Espressif PCN20251201 moves ESP32-C5-WROOM-1U-N8R8 from chip revision v1.0 to v1.2 under the same part name. A module sold as plain N8R8 can carry either revision, so ask the seller which one it is. The change fixes a PSRAM consistency issue and an SRAM corruption issue. LCSC C51950748 (plain N8R8) and C54951858 (N8R8-V1.2) use the same EasyEDA footprint record, and the PCN describes a chip metal-layer change with the same part number, so both fit U6. The module marking tells them apart: MCXXXX is v1.0, MDXXXX is v1.2. Do not set "Minimum Supported ESP32-C5 Revision" to v1.2 in firmware for a v1.0 module.

Sources: [PCN20251201](https://documentation.espressif.com/en/PCN20251201_Upgrade_of_ESP32-C5_Series_Products.html), [DigiKey listing](https://www.digikey.com.au/en/products/detail/espressif-systems/ESP32-C5-WROOM-1U-N8R8/27566801).
