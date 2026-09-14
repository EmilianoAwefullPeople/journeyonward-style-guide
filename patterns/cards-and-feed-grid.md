# Cards & Feed Grid

> Reference only. See repo [README](../README.md). Snippets are illustrative.

## Two view modes: single-column feed vs. masonry grid

v1's Home/Review screens offer two ways to browse entries, toggled by the user (icon pair: `FeedIcon` / `GridIcon` — see icons doc):

1. **Feed mode** — single column, full-bleed photo cards, Instagram-like.
2. **Grid mode** — Pinterest-style two-column masonry.

Both share the same underlying card visual language (full-bleed photo, gradient scrims, overlaid text) — they differ in layout, not skin.

## Feed-style card (full-bleed, text-over-photo)

The core card used on Home/feed and immediately after saving a capture:

- Photo fills the entire card (`aspectRatio` default `3/4`), `borderRadius: 16`, `overflow: hidden`.
- In **light mode only**, a hairline border is added around the card (dark mode relies on contrast against the dark background instead of a border).
- Two gradient scrims layered on top of the photo, not the whole card treated as one flat image:
  - Top: `rgba(0,0,0,0.55) → transparent`, covering the top 40% — ensures the top-left country label and top-right date stay legible over any photo.
  - Bottom: `transparent → rgba(0,0,0,0.62)`, covering the bottom 42% — same purpose for the title.
- Text overlay positions (27pt inset from every edge):
  - **Top-left**: country name, bold, 22pt, white, with a drop shadow (`rgba(0,0,0,0.75)`, 4px blur) for legibility on any photo.
  - **Top-right**: formatted date (e.g. "Mar 4 2026"), 13pt semibold.
  - **Bottom**: entry title, bold, 24pt. If the entry is **bookmarked**, the title renders as `GradientText` (babyBlue→royalGold→fern) instead of plain white — bookmark status becomes visually obvious without adding a separate icon badge.

```tsx
// Illustrative structure, not the real component
<View style={{ width, aspectRatio: 3/4, borderRadius: 16, overflow: 'hidden' }}>
  <Image source={photo} style={StyleSheet.absoluteFill} resizeMode="cover" />
  <LinearGradient colors={['rgba(0,0,0,0.55)', 'transparent']} style={topScrim} />
  <LinearGradient colors={['transparent', 'rgba(0,0,0,0.62)']} style={bottomScrim} />
  <Text style={countryStyle}>{country}</Text>
  <Text style={dateStyle}>{date}</Text>
  {isBookmarked ? <GradientText style={titleStyle}>{title}</GradientText> : <Text style={titleStyle}>{title}</Text>}
</View>
```

**Takeaway pattern**: metadata lives *on* the photo via scrims + shadowed text, not in a caption strip below it. Keeps cards dense and photo-forward.

## Masonry grid (Pinterest-style)

Two-column grid where card heights vary based on each photo's real aspect ratio (not cropped to a fixed square).

- **Column assignment**: shortest-column-first algorithm — walk entries in order, estimate each card's rendered height (photo aspect ratio + title height if present), and push each card into whichever column is currently shorter. Produces a naturally balanced masonry layout without a heavier layout library.
- **Card width math**: derived from available screen width (screen width minus horizontal padding), split in half minus gap — kept in sync with the feed-mode map/header width so grid and feed visually align to the same content width.
- Media placeholders for non-photo entries (video/audio) use an emoji + label on a flat surface color, not a generic broken-image icon.
- Card image radius: **16px** (not the 8px base card token — see spacing doc note on this inconsistency).
- Same bookmarked → `GradientText` title treatment as feed mode, for consistency across both view modes.
- Rendering priority hint: first 6 items load at "high" priority, next 6 "normal," rest "low" — a simple, cheap way to keep initial-viewport images snappy without full virtualization tuning.

## Placeholder card (no photo)

For video/audio/missing-media entries, instead of a gray box: render a `MeshGradientBackground` (see effects doc) behind a centered emoji + label. Keeps empty/non-photo cards visually consistent with the app's atmosphere rather than looking broken or unfinished.

## Review screen: browsing beyond a single feed

The Review tab layers more browsing structure on top of the same card language (see `screens/review-grid-filters.png`, `screens/review-trips-dark.png`):

- A 3-way segmented control switches between **Dashboard** (stats/charts), **Your Trips** (recaps/stories carousel + trip-album grid), and **All Memories** (the feed/grid views described above) — one screen, three different "lenses" on the same underlying entries rather than separate tabs.
- **All Memories** adds a search bar (title/place/date/season) and a row of filter chips (Experiences / Bookmarks / Photos / Notes) above the grid — chips use a filled-black-pill-when-active style, not underlines or checkmarks.
- **Your Trips** shows a horizontal-scroll "Recaps & Stories" carousel (gradient-bordered cards, see navigation doc) above a standard 2-column "Albums" grid; empty albums show a muted "No photos yet" placeholder rather than being hidden.
