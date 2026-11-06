# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Hardware project: ESP32 Marauder board on the ESP32-C5 with the Pancake pinout. There is no code, build, lint or test. It targets the public `MARAUDER_PANCAKE` firmware build, so the pinout must not change.

**The source of truth is the EasyEDA Pro cloud project, not this repo.** The repo holds records and exported files. Schematic and PCB can only be read or changed through a running EasyEDA Pro client, using the `easyeda-api` skill and its bridge on ports 49620–49629.

| Document | UUID |
|---|---|
| Project `Marauder C5 Pancake` | `ea209e92cc404bea925352efef413281` |
| PCB | `02c1fd8cf58bcdf4` |
| Schematic | `d32a2bca4e9acca0` |

## Layout

- `docs/requirements-and-plan.md` is the main document: requirements, plan, gate log, and decision log.
- `docs/verification.md` holds the checks, known risks, and part availability. Read it before saying anything about the state of the board.
- `release/rev1/` is the ordering package. `README.md` and `BOM.md` are written by hand or generated; the rest are EasyEDA exports. `*.step` stays local.
- This repository is the reference for the shared board layout in `references/release-template.md` of the `power-pcb-design` skill.

## Working rules

- Reply in Russian. Write all repository files in English.
- Use the `power-pcb-design` skill for design work, and log decisions in `docs/requirements-and-plan.md`.
- Never order boards or upload to a fabricator without explicit permission.
- Keep `origin` on HTTPS.
- The board is a prototype: nothing has been built or measured. State whether a result was calculated, checked in CAD, or measured.
- The user edits the board by hand in EasyEDA. Read positions before changing them, and do not overwrite their edits.

## Working on the board

After any PCB change:

1. Open the PCB document first. Calls fail with HTTP 500 when another tab, such as the 2D preview, is active.
2. Save, rebuild the four pours with `rebuildCopperRegion()`, and run `eda.pcb_Drc.check(true, false, true)`.
3. Check for dangling track ends with a script. DRC does not report them.
4. Export the Gerber, BOM, placement, and STEP files to `release/rev1/`, and update `SHA256SUMS`, `BOM.md`, and the renders in `docs/images/` if they changed. Check the time stamp inside the Gerber zip: an export can fail while the old file stays in place.

## Design constraints

- Stack-up JLC04161H-7628: L1 parts and signals, L2 solid GND, L3 3V3 pour, L4 signals. Pours exist on all four layers.
- Board 55.5 × 98 mm, view from the electronics side with the display socket at the bottom. Top layer is the electronics side. `J5`, `LED1`, and `C14` are on the bottom layer.
- GPS uses GPIO13/14, which are also the native USB pins, so USB goes through the CH343P on UART0. Do not move GPS: every firmware would need a custom build.
- Keep parts away from the board edge and the 18650 holder. `J1` and `SW1` sit at the edge by design.

## EasyEDA API behaviour (Pro 3.2.209)

- PCB coordinates are in mil, schematic coordinates in 0.01 inch.
- Read the netlist with `getNetlistFile('t','JLCEDA')`, with a schematic page open.
- `pcb_Document.autoRouting` keeps running after the 30 s bridge timeout, so an HTTP 500 does not mean it failed. Check the track count.
- Delete primitives one at a time or check the result. A batch delete can silently leave tracks behind.
- `sch_Drc.check` returns only a warning count, so the warning text has to be read in the UI.
- Board outlines need `pcb_PrimitivePolyline.create('', 11, ...)`. Single lines or arcs on layer 11 hang.
- Rule configuration writes need a config-only object in the current format. Do not change the account's default rule template.
- Rerunning a stub generator without limiting it to the moved parts adds tracks across existing routes.
- `dmt_Project.openProject` can discard unsaved changes.
