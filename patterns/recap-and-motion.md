# Journey Recap & Motion

> Reference only. See repo [README](../README.md). Snippets are illustrative.

"Journey Recap" is v1's end-of-trip slideshow — a series of full-screen data/photo slides (title card, opening stats, photo highlights, emotion data, weather/temperature/altitude data, activity graphs, closing) the user swipes or auto-advances through. It borrows heavily from the Instagram/Snapchat "Stories" interaction language.

## Story-style progress bar

Row of thin segments across the top, one per slide — the canonical "Stories" pattern:

- Each segment: 2pt tall, 2pt gap between, rounded.
- Completed slides: fully filled solid segment.
- Active slide: track at 30%-opacity white, with a solid fill that animates 0→100% width **linearly** over the slide's configured display duration, auto-advancing on completion.
- Upcoming slides: 30%-opacity track, unfilled.
- Supports `paused` (freezes the active segment's fill) for slides that require interaction (e.g. a multiple-choice slide) instead of auto-advancing on a timer.

```tsx
// Illustrative
<View style={{ flexDirection: 'row', gap: 2 }}>
  {segments.map((i) =>
    i < currentIndex ? <FilledSegment key={i} /> :
    i === currentIndex ? <ActiveSegment key={i} fillWidth={animatedProgress} /> :
    <EmptySegment key={i} />
  )}
</View>
```

**Takeaway pattern**: separating "track" from "fill" per segment, and driving fill width off a single `Animated.Value` reset on slide change, keeps the whole progress system driven by one timer per slide rather than a global stopwatch — simpler to pause/resume per-slide.

## Slide catalog (what kinds of slides existed)

Useful as a checklist of "what a recap can show," not as fixed IP to replicate 1:1:

- **Openers**: title card, opening stats, opening screen.
- **Photo moments**: beginning-of-trip photos, middle-of-trip photos, end-of-trip photos, color-palette-from-photos slide.
- **Data slides**: emotion data, activity distribution (graph), location data (including a "graphic" location variant), weather, temperature, altitude, journal data.
- **Interactive**: activity multiple-choice slide (user taps an answer rather than passively watching).
- **Templates**: a couple of slides were built as reusable "templates" (e.g. map snapshot, color palette) parameterized per trip rather than one-off screens — worth doing deliberately in v2.0 if recap-style features return: build 2–3 general templates (map, palette, stat-card) rather than N bespoke slide components.

## Motion conventions

- All-dark canvas: recap slides render on the same `#141212` near-black background as the rest of dark mode — no separate "recap theme."
- Slide titles: centered, 18pt semibold Inter, generous `marginBottom: 44` — recap typography is intentionally simpler/larger than in-app UI type, since it's meant to be glanced at, not read closely.
- Dedicated motion helpers existed for recurring recap effects — an animated title entrance, a count-up number animation (for stats like "12 countries visited"), a scale-entrance hook, and a staggered-entrance hook for lists of items appearing one after another. **Pattern worth keeping**: factor slide-entrance motion into a few shared hooks (count-up, scale-in, stagger-in) that any new slide can reuse, rather than each slide hand-rolling its own `Animated.Value` choreography.
- Respects the same **Reduce Motion** setting used elsewhere in the app (see navigation doc) — recap is not exempt from accessibility motion preferences just because it's a "fun" feature.
