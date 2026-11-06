# Release rev1

Prototype release, exported from EasyEDA Pro 3.2.209 on 2026-11-06. **Not validated and not ordered.** Unresolved items are in [`docs/verification.md`](../../docs/verification.md).

| File | Content |
|---|---|
| `gerber-rev1.zip` | Gerber and drill files, four layers. Upload this to the fab |
| `BOM.md` | Parts list with order links |
| `bom-rev1.csv` | EasyEDA BOM export, UTF-16; unpopulated parts are excluded |
| `placement-rev1.csv` | Part positions, UTF-16 |
| `assembly-rev1.step` | 3D assembly, 30 MB, kept local and not in git; export it again from EasyEDA if missing |

The order instruction and the reasons behind it are in the [root README](../../README.md#ordering).

Check before ordering: the bought 18650 holder against the footprint, the real display against [`docs/mechanics.md`](../../docs/mechanics.md), and part stock.
