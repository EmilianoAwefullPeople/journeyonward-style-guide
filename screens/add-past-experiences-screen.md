# Add Past Experiences Screen

**Route name:** `AddPastExperiences`
**File:** `src/screens/AddPastExperiencesScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
A gallery-picker entry point for backfilling old memories: lets the user pick up to 3 existing photos from their camera roll and feeds them into the *same* review/save flow the live camera uses, but sourcing the entry's timestamp/GPS/altitude from EXIF metadata instead of the current moment. Serves the Retrospective Traveler persona directly (per the file's own header comment, lines 1–4).

## Entry points & navigation
- Only entry point in the codebase: `ImportLatestScreen.tsx:621` → `navigation.navigate('AddPastExperiences')` (no params) — reached from within the Import flow, not directly from the plus-menu.
- No params received.
- **Exit — cancel from picker or back button**: `navigation.goBack()`.
- **Exit — back out of review with 0 photos left**: returns to the `'selection'` phase rather than leaving the screen (`onBack` callback, lines 106–113).
- **Exit — saved**: `navigation.reset({ index: 1, routes: [{ name: 'MainTabs' }, { name: 'CaptureSaved', params: { entryId, previewFirstPhotoUri? } }] })` (lines 115–124) — a hard reset (not `navigate`), so backing out of `CaptureSavedScreen` afterward returns to `MainTabs`, not back through this picker.

## Layout anatomy
Two phases, mutually exclusive (`phase: 'selection' | 'review'`):
1. **Selection phase** (default): back arrow (custom-positioned, absolute), centered column — large `Ionicons "image-outline"` icon, italic serif title "Add Memories From Past Adventures", pill-shaped "Tap To Select Photos" button.
2. **Review phase**: delegates entirely to `<CaptureReviewView photos={...} exifTimestamp exifLocation exifAltitude onBack onSaved />` — the shared component also used by the live-camera `CaptureScreen` flow (see that screen's doc for the review UI itself: title input, description/emotion toggles, save/draft).

## States
- `phase`: `'selection' | 'review'` — the entire screen body swaps based on this, not a sub-section toggle.
- **Permission denied**: `ImagePicker.requestMediaLibraryPermissionsAsync()` returning non-`'granted'` shows an `Alert` ("Permission Denied...") and returns early — stays on the selection phase.
- **User cancels picker**: `result.canceled` → early return, no state change, no error shown.
- **Over-limit selection**: if the OS picker somehow returns more than `MAX_PHOTOS` (3) assets, the excess is silently truncated (`result.assets.slice(0, MAX_PHOTOS)`) and an `Alert.alert('Limit', 'Max 3 photos.')` fires after the fact (lines 73–74) — note the picker itself is configured with `selectionLimit: MAX_PHOTOS`, so this path is a defensive fallback rather than the primary UX.
- **EXIF present vs. absent**: if `exif.DateTimeOriginal` is missing, falls back to `new Date().toISOString()` (i.e. "now") for the timestamp — meaning a photo without EXIF silently gets today's date rather than surfacing a warning. Location/altitude are simply `undefined` when absent (both are optional on `CaptureReviewView`).
- **Picker error**: any thrown error during `launchImageLibraryAsync` is caught, logged via `logger.error`, and shown as a generic `Alert.alert('Error', 'Failed to pick images. Please try again.')`.

## Data & context dependencies
- No `AppContext`/`ThemeContext` data reads beyond `useTheme()` for colors/dark-mode and `useSafeAreaInsets()`.
- `expo-image-picker` (`ImagePicker.requestMediaLibraryPermissionsAsync`, `launchImageLibraryAsync` with `exif: true`).
- `utils/exifGps.getDecimalCoordinatesFromExif(exif)` — converts raw EXIF GPS tags to decimal lat/lng.
- Delegates all persistence to `CaptureReviewView`'s own `onSaved` contract — this screen itself never calls `PersistenceManager` or `AppContext` create functions directly.

## Interactions
- Single tap target in selection phase (the pill button) — no drag/multi-step gestures at this screen's own level; all review-phase interactions belong to `CaptureReviewView`.
- Custom responsive scale factor for icon/text/back-button sizing (see excerpt) rather than the shared `SPACING`/`DIMENSIONS` tokens.

## Theming notes
- Fully theme-driven; `MeshGradientBackground` flips via `isDark`.
- No light/dark-specific literal colors in this file.

## Reference UI skeleton

```tsx
// source: src/screens/AddPastExperiencesScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)

if (phase === 'review') {
  // Delegates entirely to the shared review component — same one CaptureScreen uses.
  return (
    <CaptureReviewView
      photos={selectedPhotos}
      exifTimestamp={exifTimestamp}
      exifLocation={exifLocation}
      exifAltitude={exifAltitude}
      onBack={(remaining) => { /* remaining.length === 0 -> back to 'selection'; else keep reviewing fewer photos */ }}
      onSaved={(entryId, previewFirstPhotoUri) => { /* navigation.reset -> MainTabs, CaptureSaved */ }}
    />
  );
}

// Selection phase
return (
  <View style={styles.container}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

    <TouchableOpacity style={[styles.backButton, { top: safeTop + 12 * scaleFactor, left: 18 * scaleFactor }]} onPress={() => navigation.goBack()}>
      <BackArrow width={23.75 * scaleFactor} height={26.25 * scaleFactor} color={colors.textPrimary} />
    </TouchableOpacity>

    <View style={styles.content}>  {/* flex:1, centered column, paddingHorizontal: 32 */}
      <Ionicons name="image-outline" size={72} color={colors.textPrimary} style={styles.icon} />
      <Text style={[styles.title, { fontSize: 22 * scaleFactor }]}>Add Memories From Past Adventures</Text>
      <TouchableOpacity style={styles.selectButton} onPress={pickImages}>
        <Text style={[styles.selectButtonText, { fontSize: 16 * scaleFactor }]}>Tap To Select Photos</Text>
      </TouchableOpacity>
    </View>
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.background },
  backButton: { position: 'absolute', zIndex: 100, justifyContent: 'center', alignItems: 'center' },
  content: { flex: 1, justifyContent: 'center', alignItems: 'center', paddingHorizontal: 32 },
  icon: { marginBottom: 24 },
  title: { fontFamily: FONTS.playfairDisplay, fontStyle: 'italic', color: colors.textPrimary, textAlign: 'center', marginBottom: 40, maxWidth: 300 },
  selectButton: { backgroundColor: colors.surface, paddingVertical: 16, paddingHorizontal: 32, borderRadius: 30, width: '100%', maxWidth: 280, alignItems: 'center' },
  selectButtonText: { fontFamily: FONTS.inter, fontWeight: FONT_WEIGHTS.semibold, color: colors.textPrimary },
});
```

**Key technique(s):**
- The EXIF date-parsing helper (`src/screens/AddPastExperiencesScreen.tsx:44-53`) is a manual string-split parse rather than a library call, assuming EXIF's native `YYYY:MM:DD HH:MM:SS` format:
```tsx
const parseExifDate = (exifDate: string): string => {
  if (!exifDate) return new Date().toISOString();
  try {
    const [datePart, timePart] = exifDate.split(' ');
    const [year, month, day] = datePart.split(':');
    return new Date(`${year}-${month}-${day}T${timePart}`).toISOString();
  } catch (_) {
    return new Date().toISOString();
  }
};
```
- Custom `scaleFactor = Math.min(Math.max(windowWidth, 320), 430) / 390` (line 42) sizes the icon/title/back-button — clamped to a 320-430px window range, unlike `TripAlbumDetailScreen`'s unclamped version of the same idea.

## Screenshot
_Not yet captured. TODO: capture light + dark for the selection phase; the review phase is covered by whatever screenshots `CaptureReviewView`/`CaptureScreen` get, since it's the same component._

## Open questions / follow-ups
- A photo with no EXIF date silently becomes "now" (line 82) with no visual indicator distinguishing it from a photo whose real capture date genuinely was today — worth a design decision on whether the user should be told "we couldn't read this photo's date."
- `CaptureReviewView` is shared between this screen and the live-capture flow — its spec should live once (likely under `patterns/` or its own component doc) rather than being duplicated across every screen that renders it; this doc intentionally defers to wherever `CaptureScreen`'s doc covers it.
