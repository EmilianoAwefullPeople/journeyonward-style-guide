# Icons

> Reference only. See repo [README](../README.md).

v1 used two icon sources side by side:

1. **Custom line/fill SVGs** hand-matched to Figma designs (exported as standalone files in [`svg/`](./svg) — colored `currentColor` so you can restyle them per theme/context).
2. **Ionicons** (`@expo/vector-icons`) for a few standard glyphs where a custom icon wasn't worth drawing — `compass-outline`, `camera-reverse` (camera switch), `flash`/`flash-off`. These aren't exported here since they're just Ionicons name lookups — pull the equivalent from whatever icon set v2.0 uses.

Pattern worth keeping: **don't custom-draw everything.** v1 only hand-built icons where the standard set didn't have the right shape or weight; otherwise it used a well-known icon library directly. Saves design/build time and keeps common glyphs (compass, flash) instantly recognizable.

## Custom SVG inventory

All custom icons are simple, thin-stroke or flat-fill line art, default size in the 14–25px range (they're UI-chrome scale, not illustrations), default color usually white (dark-mode-first) or near-black `#141212`, and take `width`/`height`/`color` as props so the same icon reskins per theme and context. Stroke width is either 1.5 or 2 consistently — no icon mixes stroke weights.

| Icon | File | Default size | Style | Notes |
|---|---|---|---|---|
| Heart | `heart.svg` | 16×16 | stroke, 2px | Bookmark/favorite indicator |
| Back arrow | `back-arrow.svg` | 19×21 | flat fill | Nav back button |
| Calendar | `calendar.svg` | 14×14 (24×24 viewBox) | stroke, 2px | Date pickers |
| Camera | `camera.svg` | 25×24 | fill + stroke, layered lens | Capture button — has a "lens fill" concept: a light-colored inner circle so the lens reads as a cutout against a dark body |
| Close (X) | `close.svg` | 16×16 | stroke, 2px | Dismiss/modal close |
| Feed (2 bars) | `feed.svg` | 20×20 | flat fill | View-toggle: single-column feed mode |
| Grid (4 squares) | `grid.svg` | 20×20 | flat fill | View-toggle: Pinterest grid mode |
| Location pin | `location.svg` | 14×14 (12×16 viewBox) | flat fill | Classic map-pin silhouette |
| More (···) | `more.svg` | 24×24 | stroke dots, 2px | Horizontal 3-dot overflow menu |
| Pen | `pen.svg` | 18×18 | stroke, 1.6–2px | Edit/compose action |
| Shuffle | `shuffle.svg` | 20×20 | stroke, 2px | Random feed order |
| Trash | `trash.svg` | 18×18 | stroke, 1.5px | Delete action |

## Design conventions to carry forward

- **Props contract**: every icon takes `width`, `height`, `color` (all optional with sensible defaults) — makes icons trivially theme-aware and reusable at any size, rather than baking in fixed dimensions or colors.
- **View-mode toggle icons come in matched pairs** (`feed`/`grid`) drawn at the same visual weight so switching between them doesn't cause a jarring size/weight jump.
- **The camera icon deliberately supports a two-tone "cutout lens" look** (`lensFill` separate from `color`) rather than a single-color icon — worth keeping as a technique for any icon that needs to read as "a lens" or "a hole," not just a flat glyph.
