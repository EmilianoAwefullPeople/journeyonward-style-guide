# Screens

Visual reference screenshots captured live from the running v1 app (iOS Simulator, real seeded data), 2026-09-14. Pair these with the docs in `../patterns/` — the screenshots show the result, the pattern docs explain the technique.

| File | Screen | Theme | Notes |
|---|---|---|---|
| `home-feed-list.png` | Home — single-column feed mode | Light | `FeedStyleEntryCard`: full-bleed photo, gradient scrims, country/date/title overlay. See `patterns/cards-and-feed-grid.md`. |
| `home-grid-pinterest.png` | Home — Pinterest masonry grid mode | Light | Two-column masonry; also shows a **recap card** with the gradient border treatment. See `patterns/cards-and-feed-grid.md`. |
| `home-grid-dark.png` | Home — Pinterest masonry grid mode | Dark | Same layout as above under the dark palette — useful for comparing how card contrast/scrims hold up in both themes. |
| `capture-action-sheet.png` | Capture bottom sheet (opened via the `+` tab) | Light | 2×2 action grid (Live Capture / Import Memories / Create A Postcard / Journal Entry) + a gradient-bordered "Unpack A Trip" promo card. Confirms the gradient-border treatment (see `patterns/navigation-and-tab-bar.md`) is used beyond just the tab bar and capture button — it's a general "this is a special/featured card" signal. |
| `review-grid-filters.png` | Review — All Memories grid, with search + filter chips | Light | Segmented view control (stats / trips / grid) at top, search bar, filter chips (Experiences / Bookmarks / Photos / Notes) above the masonry grid. |
| `review-trips-dark.png` | Review — Your Trips tab (Recaps & Stories carousel + Albums) | Dark | Horizontal carousel of gradient-bordered recap/story cards, with a trip-albums grid below (empty-state albums show a muted "No photos yet" placeholder). |

## Not included

Journey Recap's full-screen slideshow wasn't captured — the recap experience is being redesigned from scratch for v2.0, so a v1 reference wasn't worth preserving here (see `patterns/recap-and-motion.md` for the *structural* pattern notes — progress bar, slide catalog, motion hooks — which still apply even without slide screenshots).
