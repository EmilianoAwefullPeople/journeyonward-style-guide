# Unpack Add Memories Screen

**Route name:** `UnpackAddMemories`
**File:** `src/screens/unpack/UnpackAddMemoriesScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
Stage 2 of 4 in the "Unpack A Trip" wizard: lets the user bring photos into the flow either by searching their library for a date range, or by picking specific photos manually from the system picker.

## Entry points & navigation
- **Path A**: from `UnpackTripDetailsScreen` step 4 → `navigate('UnpackAddMemories', { albumId, tripStartDate, tripEndDate, albumName, albumDescription })`.
- **Path B (photos-first)**: from `UnpackTripDetailsScreen` step 1's "Or start by importing photos" link → `navigate('UnpackAddMemories', {})` (no params; `isPhotosFirst = !albumId`).
- On confirmed selection (either path) → `navigate('UnpackProcessing', { albumId, pickedAssets, albumName, albumStartDate: tripStartDate, albumEndDate: tripEndDate, albumDescription })`.
- Back chevron → `navigation.goBack()`.

## Layout anatomy
1. `MeshGradientBackground` backdrop.
2. Header: back chevron + `UnpackStepBar` (`currentStage={isPhotosFirst ? 1 : 2}`).
3. Title "Add your memories" + subtitle.
4. Two selectable option cards: "By date range" (calendar icon) and "Pick manually" (images icon) — selecting either sets `mode`, and selecting "Pick manually" immediately triggers the system photo picker.
5. Conditional date-range panel (only when `mode === 'date_range'`): From/To date pills that toggle inline `DateTimePicker`s, and a "Find photos" button.
6. Full-screen loading overlay (mesh background + spinner), shown between the moment a selection is confirmed and the moment navigation actually occurs.

## States
- `mode: 'date_range' | 'manual' | null` — which option card is selected / which panel shows.
- `loading` — true while `MediaLibrary.getAssetsAsync`/EXIF enrichment runs for the date-range search; disables and spinners the "Find photos" button.
- `pickerLoading` — true from the moment `handleManualPick` is called until the system picker resolves or is cancelled; shows the full-screen overlay *before* the picker even opens/closes, so there's no blank-screen gap.
- `pendingAssets` — non-null once a selection (either path) is confirmed; a `useEffect` watches this and fires `navigate('UnpackProcessing', …)` on the next render tick (explicitly not inside the async handler itself — see code comment / excerpt below).
- Date-range picker sub-state: `activePicker: 'start' | 'end' | null` toggles which inline `DateTimePicker` is visible.
- No-results case: `Alert.alert('No photos found', …)` if the date-range search returns zero matches — not a persisted screen state, just a native alert.
- Permission-denied case: `Alert.alert('Permission needed', …)` for both the date-range (`MediaLibrary`) and manual (`ImagePicker`) paths.

## Data & context dependencies
- No `AppContext` reads — this screen only talks to native modules (`expo-media-library`, `expo-image-picker`) and passes results forward via nav params.
- `parseExifDateTimeToMs` (`src/utils/exifTimestamp`) — re-filters date-range results by EXIF "Date Taken" rather than trusting `MediaLibrary`'s "Date Added" filter.
- `PickedGalleryAsset` type from `src/services/ImportLatestService`.

## Interactions
- Option-card tap: select mode; for "Pick manually" this also `await`s `handleManualPick()` inline in the `onPress`.
- Date pill tap: toggles the corresponding inline date picker open/closed.
- "Find photos": runs the paginated `MediaLibrary.getAssetsAsync` query (200/page, capped at `MAX_PHOTOS = 500`), enriches each asset with EXIF via `getAssetInfoAsync` (network download disabled to avoid iCloud-hang), then re-filters by the EXIF date before committing `pendingAssets`.

## Theming notes
- Fully theme-aware; selected option cards get a 2px `colors.textPrimary` border vs. 1px `colors.border` when unselected — a border-width-swap selection affordance rather than a background-color swap.

## Reference UI skeleton
Whole render tree, business logic stripped.

```tsx
// source: src/screens/unpack/UnpackAddMemoriesScreen.tsx (skeleton — business logic stripped)
return (
  <View style={[styles.container, { backgroundColor: colors.background }]}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

    <View style={[styles.header, { paddingTop: insets.top + SPACING.sm }]}>
      <View style={styles.headerRow}>
        <TouchableOpacity style={styles.backBtn} onPress={() => navigation.goBack()}>
          <Ionicons name="chevron-back" size={22} />
        </TouchableOpacity>
      </View>
      <UnpackStepBar currentStage={isPhotosFirst ? 1 : 2} />
    </View>

    <View style={[styles.content, { paddingBottom: insets.bottom + SPACING.xl }]}>
      <AppText style={styles.title}>Add your memories</AppText> {/* Playfair Display italic bold, 26px */}
      <AppText style={styles.subtitle}>Choose how to bring in your photos.</AppText>

      <View style={styles.cards}> {/* gap: SPACING.sm */}
        {/* Date-range option — border swaps to 2px colors.textPrimary when selected */}
        <TouchableOpacity
          style={[styles.optCard, { borderColor: mode === 'date_range' ? colors.textPrimary : colors.border, borderWidth: mode === 'date_range' ? 2 : 1 }]}
          onPress={() => setMode('date_range')}
        >
          <Ionicons name="calendar-outline" size={24} />
          <View style={{ flex: 1 }}>
            <AppText style={styles.optTitle}>By date range</AppText>
            <AppText style={styles.optBody}>Photos from your trip dates are pulled in automatically.</AppText>
          </View>
        </TouchableOpacity>

        {/* Manual-pick option — tap immediately launches the system photo picker */}
        <TouchableOpacity
          style={[styles.optCard, { borderColor: mode === 'manual' ? colors.textPrimary : colors.border, borderWidth: mode === 'manual' ? 2 : 1 }]}
          onPress={async () => { setMode('manual'); await handleManualPick(); }}
        >
          <Ionicons name="images-outline" size={24} />
          <View style={{ flex: 1 }}>
            <AppText style={styles.optTitle}>Pick manually</AppText>
            <AppText style={styles.optBody}>Choose specific photos from your gallery.</AppText>
          </View>
        </TouchableOpacity>
      </View>

      {mode === 'date_range' && (
        <View style={styles.dateSection}> {/* border, borderRadius:14, padding: SPACING.md, gap: SPACING.sm */}
          <View style={styles.dateRow}>
            <AppText style={styles.dateLabel}>From</AppText>
            <TouchableOpacity style={styles.datePill} onPress={() => setActivePicker(activePicker === 'start' ? null : 'start')}>
              <AppText style={styles.datePillText}>{startDate.toLocaleDateString(...)}</AppText>
            </TouchableOpacity>
          </View>
          {activePicker === 'start' && <DateTimePicker value={startDate} mode="date" display="spinner" maximumDate={endDate} onChange={...} />}

          <View style={styles.dateRow}>
            <AppText style={styles.dateLabel}>To</AppText>
            <TouchableOpacity style={styles.datePill} onPress={() => setActivePicker(activePicker === 'end' ? null : 'end')}>
              <AppText style={styles.datePillText}>{endDate.toLocaleDateString(...)}</AppText>
            </TouchableOpacity>
          </View>
          {activePicker === 'end' && <DateTimePicker value={endDate} mode="date" display="spinner" minimumDate={startDate} maximumDate={new Date()} onChange={...} />}

          <TouchableOpacity style={[styles.findBtn, { opacity: loading ? 0.6 : 1 }]} onPress={handleDateRangeSearch} disabled={loading}>
            {loading ? <ActivityIndicator size="small" /> : <AppText style={styles.findBtnText}>Find photos</AppText>}
          </TouchableOpacity>
        </View>
      )}
    </View>

    {/* Loading overlay — see "Key technique" for why it appears before navigation fires */}
    {(pickerLoading || pendingAssets) && (
      <View style={[StyleSheet.absoluteFill, styles.pendingOverlay, { backgroundColor: colors.background }]} pointerEvents="box-only">
        <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
        <ActivityIndicator size="large" color={colors.textSecondary} />
      </View>
    )}
  </View>
);
```

**Key technique(s):** The file's own header comment explains a real iOS gotcha this screen works around — navigating while a native view controller (e.g. `PHPickerViewController`) is still mid-dismiss gets silently dropped by UIKit. So `navigate('UnpackProcessing', …)` is never called directly from the async picker callback; instead the confirmed selection is stored in `pendingAssets` state, and a `useEffect` fires the navigate after React commits the next render — always outside any native transition window — `file:79-92`. The full-screen overlay is shown in the *same* render as `setPendingAssets()`/`setPickerLoading(true)`, so the user sees a loading state immediately, before `UnpackProcessing` even mounts.

## Screenshot
_Not yet captured. TODO: capture light + dark for: initial state (no mode selected), date-range mode with a picker open, the manual-pick overlay, and the "no photos found" alert._

## Open questions / follow-ups
- `exif` is intentionally omitted from the manual `ImagePicker.launchImageLibraryAsync` call (documented in-code as avoiding a ~60s hang on large selections) and instead backfilled later in `UnpackProcessingScreen` — worth keeping this split in mind if EXIF-dependent features are ever added to this screen directly, since assets here may not have EXIF yet.
- This flow duplicates the general "import photos" concept already covered by `ImportLatestScreen`/`ImportLatestService` (see that screen's doc) but with its own date-range/EXIF-clustering logic. Worth flagging to product/eng whether these two import paths are meant to diverge long-term or should share more code.
