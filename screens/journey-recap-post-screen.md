# Journey Recap Post

**Route name:** `JourneyRecapPost`
**File:** `src/screens/JourneyRecapPostScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
The landing screen right after a recap finishes playing: turns the trip's photos into a horizontally-swipeable carousel of shareable postcards (reusing the app's postcard template system) and lets the user share/save one or replay the recap. Primarily serves the Travel Sharer persona — this is the screen that produces the actual shareable artifact.

## Entry points & navigation
- **In:** only from `JourneyRecapPlayerScreen`, automatically, when the user reaches the final slide (`navigation.navigate('JourneyRecapPost', { albumId })` inside `goNext`). Not reachable any other way.
- **Params:** `{ albumId?: string }`.
- **Out:** "Done" → resets navigation to `MainTabs (Review tab)` → `TripAlbumDetail` for that album (via `CommonActions.reset`, not a simple `goBack`, so the recap/post screens aren't left on the stack); falls back to `navigation.goBack()` if that dispatch throws. "Replay from beginning" → `navigation.navigate('JourneyRecapPlayer', { albumId })`.

## Layout anatomy
1. Top bar: "Done" (left) / "Your postcards" title (center) / empty spacer (right, balances the layout).
2. Horizontal paging `ScrollView` (`pagingEnabled`) — one postcard template per page, centered, `SW - 56` wide at a fixed `4/5` aspect ratio.
3. Dot pagination row below the carousel (only shown if `cards.length > 1`) — active dot white + scaled 1.2×, inactive dots 25%-opacity white.
4. Bottom area: full-width white pill "Share" button (shows an `ActivityIndicator` while `sharing`), and a smaller "Replay from beginning" text link underneath.

## States
- **Empty** (`!album || cards.length === 0`): centered "No postcards to show." + a "Go to your trip" link back. Triggered when the album can't be found or no card specs could be built (e.g. trip has zero usable photos).
- **Populated**: the carousel described above.
- **`sharing`**: boolean flag while `handleShare` is capturing/exporting the current card — swaps the Share button's label for a spinner and disables the button.

## Data & context dependencies
- `useAppContext()` → `entries`, `tripAlbums`.
- Builds its own card list independently of the recap player: filters the album's non-draft entries down to photo entries, **bookmarked entries sorted first**, capped at 12, then `buildRecapCards()` distributes them cyclically across a fixed, photo-count-ordered list of postcard templates (`RECAP_TEMPLATE_SPECS`: T-02 Camera Frame → T-03 Statement Card → T-11 Diptych → T-05 iMessage Thread → T-06 Triptych Stamps → T-01 Quad Collage, each requiring progressively more distinct photos), so a trip with few photos still gets at least the 1-photo templates.
- Postcard template components are lazy-loaded per-card via `loadTemplateComponent(templateId)` (local `require()` switch, same lazy-load convention as the rest of the app).
- `components/postcards/templates/diptychCaptionUtils` → `deriveDiptychLabels()` for the T-11 template's top/bottom location labels.
- `services/AnalyticsService` → `Analytics.onPostcardExported(albumId)` fired after a successful share.
- Sharing/saving: `react-native-view-shot`'s `captureRef()` to rasterize the current card, then `Share.share()` on iOS or `expo-media-library`'s `saveToLibraryAsync()` (after a permission prompt) on Android.

## Interactions
- Paging `ScrollView` with `onMomentumScrollEnd` computing the active dot/share-target index from `contentOffset.x / SW` — no Reanimated carousel library, just a plain scroll view snapped to `pagingEnabled` pages.
- `handleShare` waits two nested `requestAnimationFrame` calls before capturing, to ensure the just-scrolled-to card has fully laid out/painted before `captureRef` rasterizes it.

## Theming notes
- Same as the recap player: fixed dark canvas (`#141212` background, white/translucent-white text), not theme-aware — recap-family screens intentionally opt out of light/dark.

## Reference UI skeleton

```tsx
// source: src/screens/JourneyRecapPostScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)

// One carousel page's content — picks the T-11 Diptych's unique two-slot prop
// contract vs. every other template's uniform `slots[]` array (see Key technique).
const RecapCard = React.forwardRef(({ spec }, ref) => {
  const Component = loadTemplateComponent(spec.templateId); // require() switch — :103-131
  if (!Component) return null;
  if (spec.templateId === 'T-11') {
    return (
      <Component ref={ref}
        topPhoto={spec.photos[0] ?? null} bottomPhoto={spec.photos[1] ?? null}
        topLabel={/* deriveDiptychLabels(...) */ ''} bottomLabel={''}
        topTransform={DEFAULT_POSTCARD_PHOTO_TRANSFORM} bottomTransform={DEFAULT_POSTCARD_PHOTO_TRANSFORM}
        onTopTransformChange={NOOP} onBottomTransformChange={NOOP} onSlotPress={NOOP}
        previewMode editingChromeSuppressed aspectRatio={ASPECT_RATIO} />
    );
  }
  const slots = spec.photos.map((photoUri, i) => ({ photoUri, transform: DEFAULT_POSTCARD_PHOTO_TRANSFORM, resetKey: 0, entry: spec.entries[i] ?? null }));
  return <Component ref={ref} slots={slots} onSlotPress={NOOP} onTransformChange={NOOP}
    templateData={{}} onTemplateDataChange={NOOP} aspectRatio={ASPECT_RATIO} editingChromeSuppressed previewMode />;
});

function JourneyRecapPostScreen() {
  // photoEntries: album's non-draft photo entries, bookmarked-first, capped at 12 — :205-223
  // cards: buildRecapCards(photoEntries) — cyclically distributes photos across RECAP_TEMPLATE_SPECS — :77-100

  if (!album || cards.length === 0) {
    return (
      <View style={[styles.container, { alignItems: 'center', justifyContent: 'center' } /* '#141212' */]}>
        <Text style={{ color: 'rgba(255,255,255,0.5)', fontSize: 15 }}>No postcards to show.</Text>
        <TouchableOpacity onPress={handleBack} style={{ marginTop: 24 }}>
          <Text style={{ color: 'rgba(255,255,255,0.4)', fontSize: 14 }}>Go to your trip</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.container /* '#141212' */}>
      <View style={styles.topBar /* row, justify:'space-between', paddingTop: insets.top + 8 */}>
        <TouchableOpacity onPress={handleBack} accessibilityLabel="Done">
          <Text style={styles.doneText /* rgba(255,255,255,0.7), 15 semibold */}>Done</Text>
        </TouchableOpacity>
        <Text style={styles.title /* white, 17 semibold */}>Your postcards</Text>
        <View style={styles.spacer /* width:48, balances layout */} />
      </View>

      <View style={styles.carouselWrap /* flex:1, centered */}>
        <ScrollView horizontal pagingEnabled showsHorizontalScrollIndicator={false} onMomentumScrollEnd={handleScroll}>
          {cards.map((card, i) => (
            <View key={`${card.templateId}-${i}`} style={styles.cardPage /* width: SW, centered, paddingHorizontal:28 */}>
              <View style={styles.templateWrap /* width: SW-56, aspectRatio: 4/5 */}>
                <RecapCard ref={(el) => { cardRefs.current[i] = el; }} spec={card} />
              </View>
            </View>
          ))}
        </ScrollView>

        {cards.length > 1 && (
          <View style={styles.dotsRow /* row, centered, gap:6, paddingVertical:16 */}>
            {cards.map((_, i) => (
              <View key={i} style={[styles.dot /* 6x6, borderRadius:3 */, {
                backgroundColor: i === currentCard ? '#FFFFFF' : 'rgba(255,255,255,0.25)',
                transform: [{ scale: i === currentCard ? 1.2 : 1 }],
              }]} />
            ))}
          </View>
        )}
      </View>

      <View style={styles.bottomArea /* paddingHorizontal: SPACING.xl, paddingBottom: insets.bottom + 24, centered, gap:14 */}>
        <TouchableOpacity style={styles.shareButton /* full-width white pill, paddingVertical:15, borderRadius:32 */}
          onPress={handleShare} disabled={sharing} accessibilityLabel="Share postcard">
          {sharing ? <ActivityIndicator color="#141212" /> : <Text style={styles.shareButtonText /* '#141212', 16 semibold */}>Share</Text>}
        </TouchableOpacity>
        <TouchableOpacity onPress={handleReplay} accessibilityLabel="Replay from beginning">
          <Text style={styles.replayText /* rgba(255,255,255,0.45), 14 */}>Replay from beginning</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}
```

**Key technique(s):**
- The template-selection algorithm (`buildRecapCards`, :77-100) cyclically distributes a capped, bookmark-prioritized photo set across `RECAP_TEMPLATE_SPECS`, ordered fewest-photos-first, so even a sparse trip gets at least the 1-photo templates.
- `RecapCard` (:137-191) branches on `templateId === 'T-11'` because the Diptych template alone takes `topPhoto`/`bottomPhoto` props instead of the uniform `slots[]` array every other template (`T-01`/`T-02`/`T-03`/`T-05`/`T-06`) accepts — a legacy prop-contract inconsistency, not a design choice to replicate.
- `handleShare` (:239-273) waits two nested `requestAnimationFrame`s before `captureRef`, so the just-scrolled-to card has fully painted before rasterizing.

## Screenshot
_Not yet captured. TODO: capture light... actually N/A (dark-only screen) — capture the populated carousel (mid-swipe, showing dots), the Share button's loading spinner state, and the empty state._

## Open questions / follow-ups
- Not referenced by `patterns/recap-and-motion.md` at all — that doc's scope stops at the slideshow itself and doesn't mention this postcard-carousel epilogue screen. Worth folding a short note into the pattern doc (or a new `patterns/postcards.md`) since this screen is really where the postcard-template system and the recap system intersect.
- The `RECAP_TEMPLATE_SPECS` ordering ("sorted by photo count, fewest first, so the carousel starts with single-photo cards") is a deliberate UX choice specific to this screen — different from however templates are chosen in `PostcardTemplatesScreen`/`PostcardEditorScreen` (manual template picking) or `MemoriesSlideshowScreen` (exact-fit collage picking, see that doc) — three different template-selection strategies exist across the app and aren't unified.
