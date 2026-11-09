# Release rev1

Prototype, exported from EasyEDA Pro 3.2.209 on 2026-11-07. BOM and placement exported again on 2026-11-09 after the passives change; Gerber unchanged. **Not validated and not ordered.**

| File | Content |
|---|---|
| `gerber-rev1.zip` | Gerber and drill, 4 layers. Upload this to the fab |
| `BOM.md` | Parts list with order links |
| `bom-rev1.csv` | EasyEDA BOM export, 67 fitted parts, UTF-16 |
| `placement-rev1.csv` | Part positions, UTF-16 |
| `SHA256SUMS` | Checksums |
| `assembly-rev1.step` | 3D assembly, 30 MB, local only; export it from EasyEDA if missing |

The order instruction is in the [root README](../../README.md#ordering).

## Checks on these files

- DRC, strict, after pour refill: 0 errors.
- Gerber compared with the previous export: only silkscreen changed.

## Not verified

- The bought 18650 holder against the footprint, the real display against [`docs/SPEC.md`](../../docs/SPEC.md#6-mechanics), part stock.
- 3D fit with the display and cell.
- Open items: [`docs/DESIGN-RECORD.md`](../../docs/DESIGN-RECORD.md#known-risks-for-the-first-board).
