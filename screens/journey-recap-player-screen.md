# Journey Recap Player

**Route name:** `JourneyRecapPlayer`
**File:** `src/screens/JourneyRecapPlayerScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
The actual end-of-trip slideshow: builds a sequence of data/photo slides for one trip album and auto-advances through them Stories-style. This is the feature `patterns/recap-and-motion.md` documents structurally. Serves Reflective Traveler (looking back) and Travel Sharer (generates the postcards handed to `JourneyRecapPostScreen` next) personas.

## Entry points & navigation
- **In:** many call sites pass `{ albumId }` directly (no `JourneyRecapCandidates` step — see that screen's doc): `ReviewScreen`, `HomeScreen`, `TripAlbumDetailScreen`, `StudioJourneyRecapRow` (a Studio-tab component), `JourneyRecapPostScreen` ("Replay from beginning"), `JourneyRecapCandidatesScreen` (if ever reached), and `unpack/UnpackUnpackingScreen` (auto-navigates here ~300ms after an "unpack" finishes processing — see Unpack flow docs).
- **Params:** `{ albumId: string }` (`route.params?.albumId`).
- **Out:** swiping/tapping through to the last slide → `navigation.navigate('JourneyRecapPost', { albumId })`. Back arrow → `navigation.goBack()` (also fires an analytics "partial view" event first, see below).

## Layout anatomy
Full-bleed, no safe-area-respecting content container — everything is absolutely positioned so slides can render edge-to-edge:
1. **Slide content layer** (`Animated.View`, fades via `slideOpacity`) — renders whichever of ~16 slide component types the current `RecapSlide.type` maps to (see catalog below).
2. **Off-screen preload layer** (`pointerEvents="none"`, positioned at `left: -2000`) — every recap postcard/image the slideshow will need, mounted invisibly so native image/map memory is warmed up before display.
3. **Map overlay** — a single persistent `MapView` instance, toggled between off-screen (`left: -SW*2`) and on-screen (`left: 0`) rather than mounted/unmounted, so tiles already loaded aren't discarded.
4. **Top bar** (`position: absolute, top: 0`) — `RecapProgressBar` (Stories-style segmented progress, see pattern doc) + back button. Rendered even during preload so the user can always back out.
5. **Left/right tap zones** (40% width each) for prev/next, disabled while `isInteractive` (see States) or still preloading.
6. **Loading overlay** — full-screen `#141212` scrim with an `ActivityIndicator`, fades out via `Animated.timing` once preload completes.

## States
- **`preloadPhase: 'loading'`** → **`'ready'`**: gates the progress bar, tap zones, and loading scrim. Completion is a race between two signals (whichever fires first wins, guarded by `preloadDoneRef` against double-firing): (a) `preloadAssetsUtil()` resolving after `expo-image` prefetches every URI into cache, or (b) a manual `loadedCountRef`/`totalImagesRef` counter driven by each off-screen image's `onLoad`/`onError`. A hard 6-second timeout (`setTimeout(fadeAndComplete, 6000)`) prevents an infinite spinner if both stall.
- **`error` / empty slides**: if `albumId` is missing, the album isn't found, it has no non-draft entries, or `buildRecapSlides` throws — shows a plain dark error screen with just a "Go back" link and a persistent top-bar back button (so the user isn't stuck).
- **`isInteractive`** (per-slide, from `currentSlide.interactive === true`, currently only the `activity_multiple_choice` slide): pauses the progress bar (`paused={isInteractive}`) and disables the tap-to-advance zones — the slide itself calls `onComplete={goNext}` once the user answers.
- Every other slide auto-advances via `RecapProgressBar`'s `onActiveSegmentComplete={goNext}` after `currentSlide.durationMs`.

## Data & context dependencies
- `useAppContext()` → `entries`, `tripAlbums`, `journals`.
- `route.params.albumId` drives everything: looked up against `tripAlbums`, cross-checked through `getRecapCandidates()` (same helper `JourneyRecapCandidatesScreen` uses) to get a `RecapCandidate`, then `buildRecapSlides(candidate, albumEntries, journals)` from `utils/recapBuilder` produces the ordered `RecapSlide[]`.
- `utils/recapPreloader`: `collectPreloadAssets(slides)`, `preloadAssets()`, `getCachedMapModule()` — a dedicated preloading layer specific to this screen.
- `services/AnalyticsService` (`Analytics.onRecapViewed(albumId, pct, completed)`) fired both on early exit (with the fraction of slides seen) and on reaching the end (`pct: 1.0, completed: true`).
- All 16 slide components + `RecapProgressBar` and `BackArrow` are lazy-loaded via `require()` inside a `useMemo`, matching the app-wide "no top-level RN Navigation/native-module imports" convention from `CLAUDE.md`.

## Interactions
- `Animated.Value`-driven cross-fade between slides (`slideOpacity`, 220ms) on every `goNext`/`goPrev`.
- Tap-left/tap-right zones (not swipe gestures) — simpler than a `PanResponder`/Gesture Handler carousel, consistent with `MemoriesSlideshowScreen`'s identical approach.
- Map recentring: `MapView` props (`scrollEnabled/zoomEnabled/rotateEnabled/pitchEnabled` all `false`) — the map is a static snapshot within the slide, not an interactive map.

## Theming notes
- Not theme-aware by design — hardcoded `#141212` background and white text throughout, matching `patterns/recap-and-motion.md`'s "All-dark canvas... no separate recap theme" note (recap always renders dark regardless of the app's light/dark setting).

## Reference UI skeleton

```tsx
// source: src/screens/JourneyRecapPlayerScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
function JourneyRecapPlayerScreen() {
  // slides/error built by buildRecapSlides() — :84-102
  // preloadPhase 'loading'|'ready' resolved by a dual-signal race — :117-157 (see Key technique)
  // mapComponents loaded lazily once preloadData.needsMap — :161-177

  if (error || slides.length === 0) {
    return (
      <View style={styles.container /* flex:1, backgroundColor:'#141212' */}>
        <View style={styles.errorWrap /* flex:1, centered, paddingTop: insets.top + 60 */}>
          <Text style={styles.errorText /* rgba(255,255,255,0.8) */}>{error ?? 'No slides to show.'}</Text>
          <TouchableOpacity onPress={handleBack}><Text style={styles.errorBack /* rgba(255,255,255,0.5) */}>Go back</Text></TouchableOpacity>
        </View>
        <View style={[styles.topBar, { paddingTop: insets.top }]}>
          <View style={styles.header}>
            <TouchableOpacity onPress={handleBack} style={styles.backButton}>
              <BackArrow color="#FFFFFF" width={24} height={24} />
            </TouchableOpacity>
          </View>
        </View>
      </View>
    );
  }

  const currentSlide = slides[currentIndex];
  const isInteractive = currentSlide?.interactive === true;

  return (
    <View style={styles.container}>
      {/* 1. Slide content — cross-fades via Animated.Value slideOpacity, 220ms (animateIn, :183-186) */}
      <Animated.View style={[styles.fill /* absoluteFillObject */, { opacity: slideOpacity }]} key={currentSlide.id}>
        {renderSlide(currentSlide) /* switch over ~16 slide.type values — :214-332, one <XSlide {...payload}/> each */}
      </Animated.View>

      {/* 2. Off-screen preload layer — mounted but invisible, warms native image/map memory */}
      {preloadData && (
        <View style={styles.offScreen /* position:'absolute', left:-2000, opacity:0 */} pointerEvents="none">
          {preloadData.postcardItems.map((item) => (
            <RecapPostcard key={item.photoUri} photoUri={item.photoUri} location={item.location}
              width={OFFSCREEN_CARD_WIDTH} onLoad={handleImageLoad} onError={handleImageLoad} />
          ))}
          {preloadData.imageUris.map((uri) => (
            <ExpoImage key={uri} source={{ uri }} style={styles.offScreenImage /* SW x SW */}
              contentFit="cover" cachePolicy="memory-disk" onLoad={handleImageLoad} onError={handleImageLoad} />
          ))}
        </View>
      )}

      {/* 3. Single persistent MapView — repositioned on/off-screen rather than mounted/unmounted */}
      {preloadData?.mapProps && mapComponents && (
        <Animated.View style={[styles.mapOverlay, { left: isMapSlide ? 0 : -SW * 2, opacity: isMapSlide ? slideOpacity : 0 }]} pointerEvents="none">
          <MapView style={{ width: SW, height: MAP_HEIGHT }} initialRegion={preloadData.mapProps.region}
            scrollEnabled={false} zoomEnabled={false} rotateEnabled={false} pitchEnabled={false}>
            {preloadData.mapProps.locations.map((loc, i) => (
              <Marker key={i} coordinate={{ latitude: loc.latitude, longitude: loc.longitude }} title={loc.label} pinColor="#fed86d" />
            ))}
          </MapView>
        </Animated.View>
      )}

      {/* 4. Top bar — always rendered so back button works during loading */}
      <View style={styles.topBar /* position:absolute, top:0, zIndex:10, paddingTop: insets.top */} pointerEvents="box-none">
        {preloadPhase === 'ready' && (
          <RecapProgressBar currentIndex={currentIndex} totalCount={slides.length}
            activeSegmentDurationMs={currentSlide.durationMs}
            onActiveSegmentComplete={isInteractive ? undefined : goNext} paused={isInteractive} />
        )}
        <View style={styles.header}>
          <TouchableOpacity onPress={handleBack} style={styles.backButton} accessibilityLabel="Close recap">
            <BackArrow color="#FFFFFF" width={24} height={24} />
          </TouchableOpacity>
        </View>
      </View>

      {/* 5. Left/right 40%-width tap zones — disabled during preload or an interactive slide */}
      {preloadPhase === 'ready' && !isInteractive && (
        <>
          <TouchableOpacity style={styles.tapLeft /* absolute left, width:'40%' */} onPress={goPrev} activeOpacity={1} />
          <TouchableOpacity style={styles.tapRight /* absolute right, width:'40%' */} onPress={goNext} activeOpacity={1} />
        </>
      )}

      {/* 6. Loading overlay — fades out (200ms) once preloadPhase flips to 'ready' */}
      {preloadPhase === 'loading' && (
        <Animated.View style={[StyleSheet.absoluteFill, styles.loadingOverlay /* '#141212' */, { opacity: loadingOpacity }]}>
          <ActivityIndicator color="rgba(255,255,255,0.35)" size="large" />
        </Animated.View>
      )}
    </View>
  );
}
```

**Key technique(s):**
- The dual-signal preload-completion race (`fadeAndComplete`/`handleImageLoad`, :117-157) — whichever of `expo-image` prefetch resolving or the manual `onLoad`/`onError` counter fires first wins, guarded by `preloadDoneRef`, with a hard 6s timeout ceiling.
- All 16 slide components + `RecapProgressBar`/`BackArrow`/`RecapPostcard` are lazy-`require()`'d inside one `useMemo` (`:58-82`, the `C` object) — the app-wide native-module-init-safety pattern from `CLAUDE.md`, applied to a large component set at once.
- The single persistent `MapView` (:161-177, :477-511) is repositioned rather than mounted/unmounted so already-loaded tiles survive slide changes — the same "keep it mounted, move it" trick as the off-screen preload layer.

## Screenshot
_Not yet captured. TODO: capture at least one representative slide of each catalog type in the list below (title card, opening stats, a photo slide, emotion data, the graphic-location/map slide, the interactive activity-multiple-choice slide), plus the loading overlay and the error state._

## Open questions / follow-ups
- **Slide catalog is larger than `patterns/recap-and-motion.md` documents.** The pattern doc's checklist (openers, photo moments, data slides, interactive, templates) still holds structurally, but the live `switch` in `renderSlide` has ~16 concrete slide types (`opening_screen`, `title_card`, `opening_stats`, `beginning_of_trip_photos`, `activity_multiple_choice`, `activity_distribution_graph`, `color_palette`, `color_palette_photo`, `emotion_data`, `journal_data`, `location_data`, `graphic_location_data`, `middle_of_trip_photos`, `weather_data`, `temperature_data`, `altitude_data`, `end_of_trip_photos`) — worth updating the pattern doc's catalog to this exact list if it's meant to stay accurate, or leaving it as-is if the doc is deliberately kept abstract.
- The pattern doc's "template" note (map snapshot, color palette as reusable templates) matches: `color_palette` renders `templates/T7ColorPalette`, distinct from the one-off slide components under `components/recap/slides/`.
- `RecapPostcard` (used in the off-screen preload layer) is the same component family referenced by `JourneyRecapPostScreen`'s postcard templates — the recap player and the post-recap postcard carousel share visual DNA worth documenting together if a designer revisits either.
