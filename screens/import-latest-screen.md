# Import Latest Screen

**Route name:** `ImportLatest`
**File:** `src/screens/ImportLatestScreen.tsx` (1,501 lines — the largest screen file in the app)
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
Bulk-imports existing camera-roll photos into the app as `Entry` records, either by scanning a recent time window (last 24h/72h/custom range) or by letting the user hand-pick any photos from their library. Suggestions are grouped into "moments" (multi-photo experiences) automatically. Serves the Retrospective Traveler persona at scale — this is the many-at-once counterpart to `AddPastExperiencesScreen`'s one-at-a-time flow (which this screen links out to, see below).

## Entry points & navigation
- Reached from the tab bar's "+" plus-menu → "Import Memories" (`handlePlusMenuAction('ImportLatest')` in `AppNavigator.tsx`).
- No route params.
- "Or add a past experience manually" link (filter screen) → `navigation.navigate('AddPastExperiences')`.
- Back button: aborts any in-flight scan/save (`importScanAbortRef`/`importSaveAbortRef`) then `navigation.goBack()`; wrapped in try/catch since `goBack()` can throw if there's nowhere to go.
- On completion ("done" state): "View on Home" / "Back to Home" both call `navigation.navigate('MainTabs', { screen: 'Home' })` (falling back to `goBack()` if that throws).
- A `navigation.addListener('beforeRemove', ...)` hook (lines 100–103) aborts both in-flight controllers if the user leaves mid-scan/mid-save via any navigation action, not just the back button.

## Layout anatomy
Six mutually-exclusive top-level renders driven by `state: 'filter' | 'loading' | 'list' | 'saving' | 'done' | 'error'` (a single `if (state === ...) return (...)` chain, not a switch):

1. **`filter`** (initial): header + "Choose a filter" with two mode buttons (**Date** / **Gallery**, `syncMode`). Date mode reveals quick-filter chips (Last 24h / Last 72h / Select date range) plus, for "custom", two stacked native `DateTimePicker` spinners; a "Scan photos" CTA is disabled (`isScanDisabled`) until a valid mode+range is chosen. Gallery mode shows a one-line hint and a "Select photos" CTA that immediately opens the native picker. Bottom: the manual-add link to `AddPastExperiences`.
2. **`loading`**: header + centered `LoadingSpinner`, a progress label, a horizontal progress bar (`progressFill` width % driven by `progressPercent`), and a `completed/total` detail line.
3. **`error`**: header + alert icon + `errorMessage` + gradient "Back" button.
4. **`list`** (non-empty): header (title + "Change filter" action) → `FlashList` (virtualized) of `SuggestionCard`s with a sticky `ListHeaderComponent` (`renderBatchControls`: moment/selected/photo counts + "Select all"/"Deselect all") → conditional sticky footer "Add N moments · M photos" (only when `acceptedCount > 0`).
5. **`list`** (empty after scan, `emptyAfterScan`): header + "No recent moments in {range}" + "Change filter" + gradient "Back" button — a distinct sub-branch of the `list` state, not its own `ScreenState` value.
6. **`saving`**: header-less, just centered spinner + progress label.
7. **`done`**: header + checkmark icon + "Done" + result message (varies with `failedCount`) + optional schema-failure notice + optional duplicate-skip count + "View on Home" / "Back to Home".

## States
- **`syncMode`**: `null | 'date' | 'gallery'` — selecting "Gallery" immediately triggers `handlePickFromGallery()` (opens the OS picker right away rather than requiring a second tap); selecting "Date" just reveals the quick-filter UI.
- **`isAwaitingGallery`**: shows a full-screen spinner overlay *while still on the filter screen*, activated **before** the picker opens (deliberately, per the comment at lines 367–369) to cover the gap between the native sheet dismissing and the JS promise resolving with EXIF data.
- **Scan progress**: `progressPercent`/`progressLabel`/`progressDetail`, streamed via `onProgress` callback from `discoverExperienceSuggestions` (from `ExperienceImportService`/`ImportLatestService`); `progressDetail` only shows a `completed/total` count when `progress.total > 1` or `progress.phase === 'fetch'` (line 180–182).
- **Custom range validation**: `isCustomRangeInvalid` (start > end) and a generic `filterError` string both gate the "Scan photos" button and render an inline error line; custom start/end are normalized to midnight/end-of-day respectively before scanning (lines 242–245).
- **Per-suggestion accept/skip**: `acceptedIds: Set<string>` (defaults to *all* suggestions accepted, line 197/431) — "skip" removes the suggestion from the list entirely (not just deselects it), while "accept/unaccept" is a toggle.
- **Save result**: `addedCount`, `failedCount`, `skippedDuplicateCount`, `schemaFailureNotice` — all surfaced distinctly on the `done` screen; a partial failure (`failedCount > 0`) changes the headline message but doesn't change the icon/color (still the green checkmark).
- **Abort handling**: `isImportLatestAbortedError(e)` is checked separately from other errors in both scan paths — an aborted scan (e.g. user backed out) silently returns without setting `state: 'error'`, whereas any other thrown error does.
- **`SuggestionCard` sub-state**: each card has its own photo pager (`currentImageIndex`, horizontal `FlatList`) with a staggered fade-in on mount (`createStaggeredDelay(index, 60)` — 60ms per card index) and a defensive `useEffect` (lines 897–906) that resets the pager if `imageUris` shrinks below the current index (e.g. after a re-render with fewer displayable images).

## Data & context dependencies
- `useAppContext()`: `createEntry`, `refreshData`, `tripAlbums`, `entries` (the last two passed to `persistSelectedExperiences` for dedup/matching against existing data).
- `services/ExperienceImportService`: `discoverExperienceSuggestions` (scan), `persistSelectedExperiences` (save).
- `services/ImportLatestService`: types (`ImportLatestProgress`, `ImportLatestRange`, `PickedGalleryAsset`, `SuggestedExperienceWithDisplayUris`) and `isImportLatestAbortedError`.
- `utils/dateUtils`: `formatEntryDate`, `reverseGeocode`.
- `utils/concurrency.mapWithConcurrency` — used to batch-geocode suggestion coordinates 4-at-a-time, deduped by a `lat.toFixed(2),lng.toFixed(2)` bucket key so nearby suggestions share one reverse-geocode call (lines 111–154) — a deliberate rate-limit/perf guard against bursting the OS geocoder.
- `expo-image-picker` for the gallery-selection path.
- `@shopify/flash-list` (`FlashList`) for the suggestion list — the only screen in this batch of 7 using FlashList instead of `ScrollView`/`FlatList`, presumably for large-gallery-import performance.

## Interactions
- Native `DateTimePicker` spinners for custom range.
- Horizontal photo pager per suggestion card (`FlatList` + `onMomentumScrollEnd` → `handleImagePagerMomentumEnd` computing page index from scroll offset).
- Per-card staggered opacity fade-in (`Animated.timing`, `ANIMATION_CONFIG` shared easing/duration constants from `utils/animations`).
- Abortable async work throughout via `AbortController` refs — both the scan and the save are cancellable mid-flight, including automatically on screen-leave.

## Theming notes
- Fully `useTheme().colors`-driven; the only literal colors are the 3-stop `LinearGradient` on primary CTAs (`COLORS.babyBlue, COLORS.royalGold, COLORS.fern`), reused identically on the error screen's "Back" button and the empty-list "Back" button — the same "featured/gradient" visual treatment noted elsewhere in `patterns/navigation-and-tab-bar.md` for special CTAs.
- `MeshGradientBackground` present on every state's screen except `saving` (which omits even the header) — worth confirming that's intentional minimalism during the blocking save, not an oversight.

## Reference UI skeleton
This screen is a single `if (state === ...) return (...)` chain (not a switch) — each branch below is a fully separate returned tree, collapsed here to just its distinguishing content since `styles.container`/`styles.header`/`MeshGradientBackground` are identical boilerplate across all 6.

```tsx
// source: src/screens/ImportLatestScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)

// Shared shell repeated at the top of every branch below:
// <View style={styles.container}>
//   <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
//   <View style={styles.header}>
//     <TouchableOpacity onPress={handleBack} style={styles.backButton}><BackArrow .../></TouchableOpacity>
//     <Text style={styles.headerTitle}>Import latest</Text>
//   </View>
//   ...state-specific body...
// </View>

if (state === 'filter') {
  return (
    <View style={styles.container}>
      {/* ...shared header... */}
      <ScrollView style={styles.filterScroll} contentContainerStyle={styles.filterContent}>
        <Text style={styles.filterTitle}>Choose a filter</Text>
        <View style={styles.modeFilterGrid}>
          <TouchableOpacity style={[styles.modeFilterButton, syncMode !== null && !isDateMode && styles.modeFilterButtonInactive]} onPress={() => setSyncMode('date')}>
            <Ionicons name="calendar-outline" size={18} color={colors.textPrimary} />
            <Text style={styles.filterOptionText}>Date</Text>
          </TouchableOpacity>
          <TouchableOpacity style={[styles.modeFilterButton, syncMode !== null && !isGalleryMode && styles.modeFilterButtonInactive]} onPress={() => { setSyncMode('gallery'); handlePickFromGallery(); }}>
            {/* selecting Gallery opens the OS picker immediately — no second tap */}
            <Ionicons name="images-outline" size={18} color={colors.textPrimary} />
            <Text style={styles.filterOptionText}>Gallery</Text>
          </TouchableOpacity>
        </View>

        {isDateMode && (
          <View style={styles.filterDetailSection}>
            <View style={styles.filterOptions}>
              {quickOptions.map((option) => (  // Last 24h / Last 72h / Select date range
                <TouchableOpacity key={option.id} style={[styles.filterOption, { borderColor: isSelected ? colors.textPrimary : colors.border }]} onPress={() => setSelectedFilter(option.id)}>
                  <Text style={styles.filterOptionText}>{option.label}</Text>
                </TouchableOpacity>
              ))}
            </View>
            {selectedFilter === 'custom' && (
              <View style={styles.customDateContainer}>
                <Text style={styles.customDateLabel}>Start date</Text>
                <DateTimePicker value={customStartDate} mode="date" display={Platform.OS === 'ios' ? 'spinner' : 'default'} onChange={/* setCustomStartDate */} />
                <Text style={styles.customDateLabel}>End date</Text>
                <DateTimePicker value={customEndDate} mode="date" minimumDate={customStartDate} onChange={/* setCustomEndDate */} />
              </View>
            )}
            {(filterError || isCustomRangeInvalid) && <Text style={styles.filterErrorText}>{filterError || 'Start date must be before end date.'}</Text>}
            <TouchableOpacity style={[styles.scanButton, isScanDisabled && styles.scanButtonDisabled]} onPress={handleStartScan} disabled={isScanDisabled}>
              <Text style={styles.scanButtonText}>Scan photos</Text>
            </TouchableOpacity>
          </View>
        )}

        {isGalleryMode && (
          <View style={styles.filterDetailSection}>
            <Text style={styles.galleryHint}>Pick any photos from your library. They'll be grouped into experiences by time and location.</Text>
            <TouchableOpacity style={styles.scanButton} onPress={handlePickFromGallery}><Text style={styles.scanButtonText}>Select photos</Text></TouchableOpacity>
          </View>
        )}

        <TouchableOpacity style={styles.addManuallyLink} onPress={() => navigation.navigate('AddPastExperiences')}>
          <Text style={styles.addManuallyLinkText}>Or add a past experience manually</Text>
        </TouchableOpacity>
      </ScrollView>
      {isAwaitingGallery && (  // full-screen spinner while STILL on the filter screen, shown before the OS picker even opens — see Key technique(s)
        <View style={styles.galleryOverlay}><LoadingSpinner size="large" /></View>
      )}
    </View>
  );
}

if (state === 'loading') {
  return (
    <View style={styles.container}>
      {/* ...shared header, title "Import latest"... */}
      <View style={styles.centerContent}>
        <LoadingSpinner size="large" />
        <Text style={styles.loadingText}>{progressLabel || `Scanning ${activeRangeLabel}…`}</Text>
        <View style={styles.progressTrack}><View style={[styles.progressFill, { width: `${progressPercent}%` }]} /></View>
        <Text style={styles.progressText}>{formatImportProgressPercent(progressPercent)} {progressDetail}</Text>
      </View>
    </View>
  );
}

if (state === 'error') {
  return (
    <View style={styles.container}>
      {/* ...shared header... */}
      <View style={styles.centerContent}>
        <Ionicons name="alert-circle-outline" size={48} color={colors.textSecondary} />
        <Text style={styles.errorText}>{errorMessage}</Text>
        <TouchableOpacity style={styles.primaryButton} onPress={handleBack}>
          <LinearGradient colors={[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]} start={{ x: 0, y: 0 }} end={{ x: 1, y: 0 }} style={[StyleSheet.absoluteFillObject, { borderRadius: 12 }]} />
          <Text style={styles.primaryButtonText}>Back</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

if (state === 'done') {
  return (
    <View style={styles.container}>
      {/* ...shared header, back button now goes Home... */}
      <View style={styles.centerContent}>
        <Ionicons name="checkmark-circle-outline" size={48} color={colors.fern} />
        <Text style={styles.doneTitle}>Done</Text>
        <Text style={styles.doneMessage}>{failedCount > 0 ? `Added ${addedCount} experiences. ${failedCount} failed.` : `Added ${addedCount} experiences.`}</Text>
        {schemaFailureNotice.length > 0 && <Text style={styles.doneMessage}>{schemaFailureNotice}</Text>}
        {skippedDuplicateCount > 0 && <Text style={styles.doneMessage}>{skippedDuplicateCount} duplicates skipped.</Text>}
        <TouchableOpacity style={styles.primaryButtonFlat} onPress={handleViewHome}><Text style={styles.primaryButtonFlatText}>View on Home</Text></TouchableOpacity>
        <TouchableOpacity style={styles.secondaryButton} onPress={handleBackToHome}><Text style={styles.secondaryButtonText}>Back to Home</Text></TouchableOpacity>
      </View>
    </View>
  );
}

if (state === 'saving') {
  // No header at all in this branch — not even a back button, see Open questions.
  return (
    <View style={styles.container}>
      <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
      <View style={styles.centerContent}><LoadingSpinner size="large" /><Text style={styles.loadingText}>{progressLabel}</Text></View>
    </View>
  );
}

// state === 'list', emptyAfterScan sub-branch (suggestions.length === 0 — not its own ScreenState value)
if (emptyAfterScan) {
  return (
    <View style={styles.container}>
      {/* ...shared header... */}
      <View style={styles.centerContent}>
        <Text style={styles.emptyText}>No recent moments in {activeRangeLabel}.</Text>
        <TouchableOpacity style={styles.secondaryButton} onPress={handleChangeFilter}><Text style={styles.secondaryButtonText}>Change filter</Text></TouchableOpacity>
        <TouchableOpacity style={styles.primaryButton} onPress={handleBack}>{/* same gradient "Back" as error state */}</TouchableOpacity>
      </View>
    </View>
  );
}

// state === 'list', non-empty
return (
  <View style={styles.container}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
    <View style={styles.header}>
      <TouchableOpacity onPress={handleBack} style={styles.backButton}><BackArrow .../></TouchableOpacity>
      <Text style={styles.headerTitle}>Import latest</Text>
      <TouchableOpacity onPress={handleChangeFilter} style={styles.headerActionButton}><Text style={styles.headerActionText}>Change filter</Text></TouchableOpacity>
    </View>
    <FlashList
      data={suggestions}
      keyExtractor={(item) => item.id}
      renderItem={({ item: suggestion, index }) => (
        <SuggestionCard suggestion={suggestion} index={index} isAccepted={acceptedIds.has(suggestion.id)} onAccept={...} onSkip={...} />
      )}
      ListHeaderComponent={() => (
        <View style={styles.batchControls}>
          <Text style={styles.batchSummary}>{suggestions.length} moments{acceptedCount > 0 ? ` · ${acceptedCount} selected · ${totalPhotosInAccepted} photos` : ''}</Text>
          <TouchableOpacity onPress={allSelected ? handleDeselectAll : handleSelectAll} style={styles.batchBtn}>
            <Text style={styles.batchBtnText}>{allSelected ? 'Deselect all' : 'Select all'}</Text>
          </TouchableOpacity>
        </View>
      )}
    />
    {acceptedCount > 0 && (
      <View style={[styles.footer, { paddingBottom: insets.bottom + SPACING.md }]}>
        <TouchableOpacity style={styles.addButton} onPress={handleAddExperiences}>
          <Text style={styles.addButtonText}>Add {acceptedCount} moments · {totalPhotosInAccepted} photos</Text>
        </TouchableOpacity>
      </View>
    )}
  </View>
);

// SuggestionCard — one FlashList row (src/screens/ImportLatestScreen.tsx:849-1044)
function SuggestionCard({ suggestion, index, isAccepted, onAccept, onSkip, colors, formatDate, placeLabel }) {
  return (
    <Animated.View style={[cardStyles.card, { borderColor: colors.border, opacity }]}>  {/* opacity animates 0->1, staggered by index*60ms */}
      <View style={cardStyles.mediaShell} onLayout={/* measures own width for the pager below */}>
        {imageUris.length > 0 ? (
          <FlatList
            ref={mediaFlatListRef}
            data={imageUris}
            horizontal pagingEnabled
            renderItem={({ item: uri }) => (
              <View style={[cardStyles.mediaItem, { width: mediaWidth }]}><Image source={{ uri }} style={cardStyles.feedMediaImage} resizeMode="cover" /></View>
            )}
            onMomentumScrollEnd={handleImagePagerMomentumEnd}
          />
        ) : ( /* placeholder tile when no displayable URI */ )}
      </View>
      {/* ...place/date caption row, accept/skip action row (not shown here — see cited line range) */}
    </Animated.View>
  );
}
```

**Key technique(s):**
- The geocode-request batching/dedup (`src/screens/ImportLatestScreen.tsx:117-135`) is the least-obvious technique in the file — buckets suggestions into ~1.1km grid cells so nearby suggestions share one reverse-geocode call instead of one-per-suggestion:
```tsx
const bucket = `${s.latitude.toFixed(2)},${s.longitude.toFixed(2)}`; // ~1.1km buckets
// ...then mapWithConcurrency(uniqueBuckets, 4, reverseGeocode) — one call per bucket, not per suggestion
```
- `isAwaitingGallery` (filter branch) shows its spinner overlay *while still on the filter screen*, activated **before** the native picker even opens, specifically to cover the gap between the OS sheet dismissing and the JS promise resolving with EXIF data (lines 367-369) — an easy state to miss since it doesn't correspond to a `ScreenState` value at all.

## Screenshot
_Not yet captured. TODO: capture light + dark for: filter (date mode + gallery mode), custom date range with both pickers open, loading with progress bar mid-fill, the suggestion list with a multi-photo card mid-swipe, the batch-controls header, the sticky "Add N moments" footer, empty-after-scan, error, and done (both success-only and partial-failure message variants)._

## Open questions / follow-ups
- `emptyAfterScan` (line 737) is a runtime branch inside the `list` state rather than its own `ScreenState` value — functionally fine, but means anyone tracing state transitions from the `ScreenState` type alone will miss this branch; worth noting for anyone instrumenting analytics by state name.
- The `saving` screen (lines 724–733) has no header/back option at all — confirm this is intentional (prevent interrupting a save) rather than a missing back button, since every other state has one.
- This screen and `AddPastExperiencesScreen` both import photos with EXIF handling but appear to duplicate a good amount of "extract date/GPS from EXIF" logic across `services/ExperienceImportService`, `services/ImportLatestService`, and `utils/exifGps` — not a doc issue, but worth a code-level dedup pass separately from this documentation effort.
