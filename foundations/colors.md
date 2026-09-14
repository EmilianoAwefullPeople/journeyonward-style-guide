# Colors

> Reference only — see the repo [README](../README.md) for the "no gospel" framing. Re-evaluate every value against v2.0's actual brand direction before reusing.

Journey Onward v1 uses two color layers: a small **brand/UI palette** (buttons, text, chrome) and a separate **"Organic Mesh" palette** used specifically for atmospheric gradient backgrounds. They share a few hues (fern, royal gold, baby blue) but are otherwise distinct — the mesh palette is deliberately looser and more saturated for the "tactile, sophisticated, fluid" background aesthetic, while the UI palette is flatter and more utilitarian.

Machine-readable version: [`tokens/colors.json`](../tokens/colors.json).

## Brand / UI palette

| Token | Hex | Used for |
|---|---|---|
| `lime800` | `#3A6B33` | Primary accent (dark green) |
| `splashBackground` | `#171717` | Native splash / bootstrap placeholder |
| `neutral900` | `#141212` | Main dark-mode app background |
| `white` | `#FFFFFF` | — |
| `black` | `#141212` | (yes — "black" is the same near-black as neutral900, not pure `#000`) |
| `zinc700` | `#2E2E2E` | Inactive tab text, borders |
| `zinc900` | `#1A1A1A` | Darker surface backgrounds |
| `neutral400` | `#9CA3AF` | Placeholder text |
| `stone500` | `#78716C` | Camera placeholder background |
| `stone600` | `#696969` | Inactive text, camera placeholder |
| `emotionGray` | `#A9A9A9` | Emotion slider — inactive state |
| `red600` | `#DC2626` | Video recording indicator |
| `redRecording` | `#D22121` | Recording inner circle (video/audio) |
| `zinc300` | `#D4D4D8` | Capture button outer circle |
| `buttonWhite` | `#D9D9D9` | Light gray button border tone |
| `buttonGray` | `#363636` | Button background/text |
| `infoTextGray` | `#454545` | Secondary helper text |

### Live Aura accents (also the tab-indicator gradient)

| Token | Hex | Notes |
|---|---|---|
| `babyBlue` | `#8dc0e0` | Cool accent |
| `royalGold` | `#fed86d` | Warm accent / "light leak" |
| `fern` | `#487525` | Green accent |

These three are the same three stops used in the animated tab-bar underline gradient (`babyBlue → royalGold → fern`, left to right) and in `GradientText` — see `patterns/navigation-and-tab-bar.md`.

## Semantic theme tokens (light/dark)

The app never follows system dark mode — the user explicitly picks a theme, persisted to storage. Both palettes are fully defined (not just background/text inverted); several tokens (`buttonWhite`, `buttonGray`, `stone600`, `infoTextGray`) get *different* values per theme, not just different surface colors.

| Semantic token | Dark | Light |
|---|---|---|
| `background` | `#141212` (neutral900) | `#FFFBF5` (warm off-white) |
| `surface` | `#1A1A1A` (zinc900) | `#FFFFFF` |
| `surfaceVariant` | `#222222` | `#FFF5E7` |
| `textPrimary` | `#FFFFFF` | `#141212` |
| `textSecondary` | `#9CA3AF` | `#6F665C` |
| `textTertiary` | `#2E2E2E` | `#A49685` |
| `border` | `#2E2E2E` | `#EEDFCB` |
| `borderLight` | `#D9D9D9` | `#F3E8D7` |

Notable pattern: **light mode is warm, not neutral** — background is `#FFFBF5`, borders lean tan/cream (`#EEDFCB`), not cool grays. Dark mode is a true near-black. This asymmetry (warm light / near-black dark, not just inverted grays) is a deliberate choice worth carrying forward or deliberately revisiting.

One more pattern: **`journeyCard*` tokens invert per theme** — the "journey card" component is styled with a light card in dark mode and a dark card in light mode, so it stands out against its own background rather than blending in. Worth remembering as a technique for "make this one element pop regardless of theme," even if the specific component doesn't carry over.

## Organic Mesh palette (atmospheric backgrounds only)

Used exclusively for the mesh-gradient background system (see `patterns/effects-and-atmosphere.md`). Do not use these for flat UI fills — they're tuned for gradient blending.

| Token | Hex | Role |
|---|---|---|
| `oliveLeaf` | `#344f16` | Deep base — primary dark green |
| `fern` | `#487525` | Mid-tone green |
| `royalGold` | `#fed86d` | Highlight / "light leak" (low opacity) |
| `paleSky` | `#acc6d8` | Atmospheric accent |
| `babyBlue` | `#8dc0e0` | Atmospheric accent / "light leak" (low opacity) |
| `darkSlate` | `#3e575f` | Structural depth — shadows, borders, small text |

**Rule of thumb from the source system**: `royalGold` and `babyBlue` sit on top layers at low opacity (0.2–0.4) as "light leaks"; shadows use `darkSlate` at high transparency rather than pure black; no solid strokes — edges are soft/glowing via gradient, not hard borders.
