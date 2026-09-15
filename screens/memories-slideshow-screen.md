# Memories Slideshow

**Route name:** `MemoriesSlideshow`
**File:** `src/screens/MemoriesSlideshowScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
A lighter-weight Instagram-Stories-style viewer for a trip's raw photos in chronological order — no generated data slides, no map/chart preloading, just "here are your photos, in order, with captions." The file's own header comment calls it explicitly "simpler sibling of `JourneyRecapPlayerScreen`." Serves the Retrospective/Reflective traveler flipping back through a trip quickly, as an alternative to the full data-driven recap.

## Entry points & navigation
- **In:** `ReviewScreen` and `HomeScreen`, both via `navigation.navigate('MemoriesSlideshow', { albumId: album.id })` — sit alongside (not instead of) the `JourneyRecapPlayer` entry point in both places, i.e. the user is offered both "recap" and "slideshow" as separate options for the same album.
- **Params:** `{ albumId?: string }`.
- **Out:** close (top-left `BackArrow`) → `navigation.goBack()`. Tapping the right 40%-width zone advances; on the last slide it also calls `goBack()` (the slideshow has no "post" screen equivalent — it just ends).

## Layout anatomy
1. Full-bleed slide content (`Animated.View`, cross-fades via `slideOpacity`) — either a single full-screen photo (`StoryPhoto`) or, for multi-photo entries, a postcard **collage** template.
2. Bottom gradient scrim (`transparent → rgba(0,0,0,0.75)`, 180pt tall) behind the caption.
3. Caption block (bottom, above safe area): date label + location (joined with " · "), then the entry's title/description as the caption — only rendered if at least one of the three exists.
4. Top bar: `RecapProgressBar` (same component the recap player uses) + close button.
5. Left/right 40%-width invisible tap zones for prev/next.

## States
- **Error/empty** (`error || slides.length === 0`): three sub-cases collapse into one error UI — missing `albumId` ("Missing trip."), album not found ("Trip not found."), or album has zero non-draft photo entries ("No photo memories in this trip yet.") — dark screen, message, "Go back" link, persistent close button.
- **Single-photo slide**: renders `StoryPhoto` full-screen, respecting the photo's known aspect ratio if `Entry.photoWidths`/`photoHeights` recorded it.
- **Collage slide** (`currentSlide.photoUris.length > 1`): renders a `TemplateSlot`-based collage template chosen by `pickCollageTemplate()` (see excerpt) — only if enough photos exist for an exact-fit layout; falls back silently to... actually falls through to `null` collage and would render nothing if `pickCollageTemplate` returns null for a 1-photo "collage" (shouldn't happen since `isCollage` already requires `length > 1`, and the smallest collage template needs 2 — so this path is only unreachable, not a real bug).

## Data & context dependencies
- `useAppContext()` → `entries`, `tripAlbums`.
- Builds slides directly from `Entry[]` (unlike the recap player, which goes through `getRecapCandidates`/`buildRecapSlides`): filters to non-draft, `mediaType === 'photo'` entries with at least one `photoPaths` entry, sorts by `timestamp` ascending.
- `components/postcards/templates/templateLoader` → `loadTemplateComponent(templateId)` — the **same** lazy-loader `JourneyRecapPostScreen` uses for its postcard templates, but here applied to the `C-0x` collage-only template family instead of the `T-0x` full postcard family.
- `components/review/StoryPhoto` for single-photo rendering.
- `components/recap/RecapProgressBar` — reused verbatim from the recap player (`SLIDE_DURATION_MS = 4500` here vs. per-slide `durationMs` there).

## Interactions
- Same `Animated.timing` cross-fade (220ms) and tap-left/tap-right zone pattern as `JourneyRecapPlayerScreen` — the two screens duplicate this exact motion/tap-zone logic rather than sharing a hook/component for it.
- No pinch/zoom or swipe-to-dismiss gesture — pure tap navigation.

## Theming notes
- Same fixed dark canvas as the recap family (`#141212`, white text) — not theme-aware.

## Reference UI skeleton

```tsx
// source: src/screens/MemoriesSlideshowScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
function MemoriesSlideshowScreen() {
  // slides/error built by buildSlides(albumEntries) — chronological, photo-only, non-draft — :79-113

  if (error || slides.length === 0) {
    return (
      <View style={styles.container /* '#141212' */}>
        <View style={styles.errorWrap /* flex:1, centered, paddingTop: insets.top + 60 */}>
          <Text style={styles.errorText}>{error ?? 'No slides to show.'}</Text>
          <TouchableOpacity onPress={handleClose}><Text style={styles.errorBack}>Go back</Text></TouchableOpacity>
        </View>
        <View style={styles.topBar}>
          <View style={styles.header}>
            <TouchableOpacity onPress={handleClose} style={styles.backButton} accessibilityLabel="Close">
              <BackArrow color="#FFFFFF" width={24} height={24} />
            </TouchableOpacity>
          </View>
        </View>
      </View>
    );
  }

  const currentSlide = slides[currentIndex];
  const isCollage = currentSlide.photoUris.length > 1;
  // collage = isCollage ? { CollageTemplate: loadTemplateComponent(pickCollageTemplate(...).templateId), slots } : null — :263-276

  return (
    <View style={styles.container}>
      <Animated.View style={[styles.fill /* absoluteFillObject */, { opacity: slideOpacity }]} key={currentSlide.id}>
        {collage ? (
          <View style={styles.collageWrap /* width: SW */}>
            <collage.CollageTemplate slots={collage.slots} aspectRatio={SCREEN_ASPECT_RATIO} previewMode />
          </View>
        ) : (
          <StoryPhoto uri={currentSlide.photoUris[0]} width={SW} height={SH} knownAspectRatio={currentSlide.primaryAspectRatio} />
        )}

        <LinearGradient colors={['transparent', 'rgba(0,0,0,0.75)']} style={styles.bottomScrim /* absolute bottom, height:180 */} pointerEvents="none" />

        {(currentSlide.caption || currentSlide.location || currentSlide.dateLabel) && (
          <View style={styles.captionWrap /* absolute bottom: insets.bottom + SPACING.lg, paddingHorizontal: SPACING.lg */} pointerEvents="none">
            {(currentSlide.dateLabel || currentSlide.location) && (
              <Text style={styles.dateLabel /* 13, semibold, rgba(255,255,255,0.75) */} numberOfLines={1}>
                {[currentSlide.dateLabel, currentSlide.location].filter(Boolean).join(' · ')}
              </Text>
            )}
            {currentSlide.caption && (
              <Text style={styles.caption /* 17, semibold, white */} numberOfLines={3}>{currentSlide.caption}</Text>
            )}
          </View>
        )}
      </Animated.View>

      <View style={styles.topBar /* absolute top, zIndex:10, paddingTop: insets.top */} pointerEvents="box-none">
        <RecapProgressBar currentIndex={currentIndex} totalCount={slides.length}
          activeSegmentDurationMs={SLIDE_DURATION_MS /* 4500 */} onActiveSegmentComplete={goNext} />
        <View style={styles.header}>
          <TouchableOpacity onPress={handleClose} style={styles.backButton} accessibilityLabel="Close slideshow">
            <BackArrow color="#FFFFFF" width={24} height={24} />
          </TouchableOpacity>
        </View>
      </View>

      <TouchableOpacity style={styles.tapLeft /* absolute left, width:'40%' */} onPress={goPrev} activeOpacity={1} accessibilityLabel="Previous memory" />
      <TouchableOpacity style={styles.tapRight /* absolute right, width:'40%' */} onPress={goNext} activeOpacity={1} accessibilityLabel="Next memory" />
    </View>
  );
}
```

**Key technique(s):**
- Exact-fit collage selection (`EXACT_FIT_COLLAGES`/`pickCollageTemplate`, :37-47) — picks the *largest* layout the available photo count can fully cover, so a collage slide never shows an empty "add a photo" slot or a repeated photo.
- This screen and `JourneyRecapPlayerScreen` hand-roll the identical `Animated.timing` cross-fade (220ms) + tap-zone + `RecapProgressBar` wiring independently — see that screen's doc for the same pattern; a natural shared-hook extraction if either is touched.

## Screenshot
_Not yet captured. TODO: capture a single-photo slide, a collage slide (e.g. 4-photo entry), and the caption block with all three lines (date, location, caption) populated._

## Open questions / follow-ups
- Not mentioned anywhere in `patterns/recap-and-motion.md` — worth adding a short cross-reference there ("see also: Memories Slideshow, a non-data-driven sibling") since a reader of that doc would reasonably assume it covers every Stories-style viewer in the app, and currently it only covers the recap player.
- This screen and `JourneyRecapPlayerScreen` both hand-roll the identical cross-fade + tap-zone + progress-bar wiring. If either is touched in v2.0, it's a natural extraction into one shared "story viewer" hook/component (`useStorySlides` or similar) rather than maintaining two copies.
- Confirms (again) that the app now has **three** independent template-selection strategies for postcard-style layouts: this screen's exact-fit collage picker, `JourneyRecapPostScreen`'s photo-count-ordered cyclic picker, and manual picking in `PostcardTemplatesScreen`/`PostcardEditorScreen`. Worth being deliberate about whether v2.0 wants to keep three, or unify.
