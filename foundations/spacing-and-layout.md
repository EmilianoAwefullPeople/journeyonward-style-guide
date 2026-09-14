# Spacing, Radii & Layout

> Reference only. See repo [README](../README.md).

Machine-readable version: [`tokens/spacing.json`](../tokens/spacing.json).

## Spacing scale

Standard 4pt-based scale:

| Token | Value |
|---|---|
| `xs` | 4 |
| `sm` | 8 |
| `md` | 16 |
| `lg` | 24 |
| `xl` | 32 |
| `xxl` | 48 |
| `xxxl` | 64 |

## Border radius

| Token | Value | Used for |
|---|---|---|
| `cardBorderRadius` | 8 | Base card radius (some cards override to 16 — see feed grid doc) |
| `borderRadiusMedium` | 10 | — |
| `borderRadiusLarge` | 20 | — |
| `borderRadiusModal` | 24 | Modal sheets |

Note the inconsistency worth resolving deliberately in v2.0 rather than inheriting by accident: feed/gallery cards (`PinterestGrid`, `FeedStyleEntryCard`) actually render at **16px** radius in practice, not the `cardBorderRadius` (8) token — the token undershoots what's visually used for photo cards. If you carry the token forward, reconcile this gap rather than copying it.

## Reference viewport & responsive scaling

- **Reference width: 390px** (iPhone standard / iPhone 14-ish). Several fixed dimensions (e.g. camera button size) are defined at this base width and then scaled by `screenWidth / 390` at runtime, rather than using pure percentages — gives predictable proportions across device sizes while staying responsive.
- Map/media widths are calculated as a **percentage of screen width** (e.g. 97% for the map, centered) rather than fixed pixels.
- Header height is computed relative to screen height with a floor: `max(112 * (screenHeight / 852), 100)` — scales down gracefully on shorter phones (SE) but never goes below 100pt.

## Key fixed dimensions (illustrative, not exhaustive)

| Element | Value |
|---|---|
| Bottom nav bar height | 88pt (label row only — add safe-area inset on top, see nav doc) |
| Capture button (outer / inner) | 76 / 64 |
| Camera container (at 390 base) | 344 × 463 |
| Map collapsed / expanded height | 0.608 × width / 1.282 × width (fixed aspect ratios) |

## Layout principles worth carrying forward

1. **Two-column masonry, not a fixed grid**, for photo-heavy feeds (shortest-column-first placement) — see `patterns/cards-and-feed-grid.md`.
2. **Safe-area-aware bottom chrome as a single source of truth** — v1 centralized tab-bar height + safe-area math into one small module (`mainTabBarMetrics.ts`) that every scrollable screen imports for bottom content padding, instead of each screen guessing. Worth replicating the *pattern* (one shared layout-metrics module) regardless of navigation library in v2.0.
3. **Full-bleed photo cards with text overlaid via gradient scrims**, not text-below-image — see effects/feed docs.
