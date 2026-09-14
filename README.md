# JourneyOnward Design Reference

A style and pattern reference pulled from Journey Onward v1. It exists so a designer or engineer building **v2.0** — a separate codebase — can quickly recall "what did v1 do here" for a specific piece of UI, without digging through the old repo.

## What this is

- A snapshot of colors, type, spacing, icons, effects, and layout patterns that felt right in v1.
- Organized by *design concern* (colors, feed grid, effects, navigation...) so you can jump straight to the piece you're rebuilding.
- Illustrative code: short, stripped-down snippets that show the *technique* (e.g. how the mesh gradient is layered), not production-ready modules to import.

## What this is **not**

- **Not the source of truth for v2.0.** v1's design was built under different constraints, a different product stage, and probably some decisions nobody loves anymore. Treat everything here as a candidate, not a mandate.
- **Not a component library.** Nothing here is meant to be copy-pasted wholesale into v2.0's codebase. Re-implement with v2.0's own architecture, libraries, and conventions — this just tells you what the *result* looked like and roughly how it was achieved.
- **Not exhaustive.** It covers the patterns worth reusing or at least worth knowing about, not every screen or component that ever existed in v1.

## How to use it

When you're about to build a piece of v2.0 UI, check if there's a relevant doc here first:

| Building... | Look at... |
|---|---|
| Colors, dark/light theme | `foundations/colors.md`, `tokens/colors.json` |
| Type scale, fonts | `foundations/typography.md`, `tokens/typography.json` |
| Spacing, sizing, radii | `foundations/spacing-and-layout.md`, `tokens/spacing.json` |
| Mesh/gradient backgrounds, grain, gradient text | `patterns/effects-and-atmosphere.md` |
| Feed grid, masonry layout, entry cards | `patterns/cards-and-feed-grid.md` |
| Tab bar, active-tab indicator, capture button | `patterns/navigation-and-tab-bar.md` |
| Recap slideshow / journey recap motion | `patterns/recap-and-motion.md` |
| Icons | `icons/icon-inventory.md`, `icons/svg/` |
| What a real screen looked like | `screens/` (visual reference — see its README) |

Pull the *pattern*, not the file. Adapt names, structure, and implementation details to however v2.0 is actually built.

## Source

Extracted from the Journey Onward v1 codebase (React Native / Expo) on 2026-09-14. Not kept in sync automatically — if v1's design changes after this date, this reference will drift and that's expected; it's a snapshot, not a live feed.
