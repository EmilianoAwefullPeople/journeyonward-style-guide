# Unpack Processing Screen

**Route name:** `UnpackProcessing`
**File:** `src/screens/unpack/UnpackProcessingScreen.tsx`
**Type:** Stack screen (pushed on top of tabs), gesture-disabled (no swipe-back)

## Purpose
A non-interactive transitional screen between photo selection and review: runs EXIF enrichment + moment-clustering on the picked photos in the background while showing ambient rotating copy, then silently continues the wizard once done. Exists so clustering (which can take a few seconds) never presents a blank or frozen screen.

## Entry points & navigation
- From `UnpackAddMemoriesScreen` → `navigate('UnpackProcessing', { albumId, pickedAssets, albumName, albumStartDate, albumEndDate, albumDescription })`.
- No manual exit — back navigation is disabled (per the file's header comment: "Gesture disabled: no back nav"), and there is no back button in the UI.
- On completion, navigates automatically via `navigation.dispatch(CommonActions.reset(...))` (not a plain `navigate`) to one of two next stages, **replacing** the nav stack up to `MainTabs` rather than pushing:
  - Path B (photos-first, `isPhotosFirst = !albumId`): resets to `[MainTabs, UnpackAddMemories, UnpackTripDetails]` with the newly-clustered `suggestions` and inferred date range as `UnpackTripDetails` params.
  - Path A: resets to `[MainTabs, UnpackTripDetails, UnpackHighlight]` with `suggestions` and the album metadata forwarded from the previous screen.

## Layout anatomy
1. `MeshGradientBackground` backdrop.
2. Centered content: `ActivityIndicator` (large) + a single line of fading, auto-advancing copy.

## States
- This screen has no user-facing states beyond the copy rotation — it's a single "processing" state from mount to the automatic transition.
- Internally: `copyIndex` cycles through `COPY_STEPS` (3 phrases: "Finding the moments…", "Grouping by time and place…", "Getting your memories ready…") on a fixed 1600ms/3200ms timer schedule, each transition cross-fading via `Animated.timing` on `fadeAnim`.
- EXIF-enrichment branch: only runs if `pickedAssets.some(a => a.assetId && !a.exif)` — i.e. skips re-enrichment for date-range-sourced assets (which already carry EXIF) and only backfills for manually-picked assets.
- Error handling is silent-continue: both the EXIF enrichment (`mapWithConcurrency`) and the clustering call (`runImportFromSelectedAssets`) are wrapped in try/catch that logs and falls through with an empty/partial `suggestions` array rather than surfacing an error UI — the user always reaches the next screen, possibly with fewer/no suggestions.

## Data & context dependencies
- No `AppContext` reads.
- `runImportFromSelectedAssets` (`src/services/ImportLatestService`) — the actual clustering algorithm.
- `mapWithConcurrency` (`src/utils/concurrency`) — bounds EXIF-enrichment fan-out to 8 concurrent `MediaLibrary.getAssetInfoAsync` calls.
- `inferDateRange` (local helper) — derives a start/end date from suggestion timestamps, used only for the Path-B reset.

## Interactions
None — no buttons, no gestures. The screen is designed to be inert; all transitions are timer/promise-driven.

## Theming notes
- Standard `useTheme()` background/text colors; `MeshGradientBackground` passed `lightMode={!isDark}`.

## Reference UI skeleton
The entire render tree — this screen has no conditional branches beyond the copy-rotation index; all real work (EXIF enrichment, clustering, the reset dispatch) happens in effects with no UI representation beyond the spinner + copy.

```tsx
// source: src/screens/unpack/UnpackProcessingScreen.tsx (skeleton — business logic stripped)
return (
  <View style={[styles.container, { backgroundColor: colors.background }]}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
    <View style={[styles.content, { paddingTop: insets.top, paddingBottom: insets.bottom }]}> {/* flex:1, center, gap: SPACING.xl */}
      <ActivityIndicator size="large" color={colors.textSecondary} />
      <Animated.View style={{ opacity: fadeAnim }}>
        <AppText style={styles.copy}>{COPY_STEPS[copyIndex]}</AppText> {/* 18px, centered, lineHeight:28 */}
        {/* COPY_STEPS = ['Finding the moments…', 'Grouping by time and place…', 'Getting your memories ready…'] */}
      </Animated.View>
    </View>
  </View>
);
```

**Key technique(s):** The `CommonActions.reset` used for the Path-A handoff rewrites history back to `[MainTabs, UnpackTripDetails, UnpackHighlight]` instead of pushing, so the user can never navigate "back" into the add-memories/processing screens once clustering completes — `file:135-158`. Copy rotates on a fixed 1600ms/3200ms schedule via `Animated.timing` cross-fades on `fadeAnim`, entirely decoupled from the actual clustering/EXIF-enrichment work happening in parallel — the reset fires 800ms after that work resolves, regardless of which copy phrase is showing at that moment.

## Screenshot
_Not yet captured. TODO: capture light + dark mid-copy-rotation (any of the 3 phrases is representative — the visual layout doesn't change between them)._

## Open questions / follow-ups
- The minimum visible time on this screen is soft-coded as a flat `setTimeout(run, 300)` before starting work plus an 800ms delay after clustering resolves before navigating — i.e. total transition time is `~1100ms + actual clustering time`, not tied to the copy-rotation timers (1600ms/3200ms) in any enforced way. If clustering finishes before 1600ms, the user could see the reset fire mid-first-phrase; if it takes longer than 3200ms, the copy holds on the final phrase with no further feedback that work is still ongoing. Worth checking real-world clustering latency against these constants.