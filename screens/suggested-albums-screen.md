# Suggested Albums Screen

**Route name:** `SuggestedAlbums`
**File:** `src/screens/SuggestedAlbumsScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
Auto-detects trip candidates by clustering the user's existing `Entry` records (by date/location) and lets them multi-select which detected clusters to turn into real `TripAlbum`s. Serves the Retrospective Traveler persona — someone who's already captured entries without a trip album and wants the app to retroactively organize them.

## Entry points & navigation
- Only entry point in the codebase: `ReviewScreen.tsx:117` → `navigation.navigate('SuggestedAlbums')` (no params).
- No back-arrow-and-text — back button is icon + "Back" text combined (`BackArrow` + `Text`, lines 375–376), unlike most other screens' bare icon.
- On confirming (`handleAddAlbums`) → creates one `TripAlbum` per selected suggestion via `createTripAlbum(album, 'import')`, then `navigation.goBack()`. No success screen/toast — the return to Review is the only feedback.

## Layout anatomy
1. Header: back button, title "Suggested Trips", subtitle showing count + "Tap to select" (only in `results` state).
2. Body — exactly one of three states (see States) — `loading` spinner, `empty` message, or `results` grid.
3. **Results grid**: 2-column, cards built with `cardWidth = (screenWidth - SPACING.md * 3) / 2`. Each card:
   - A square-ish (`aspectRatio: 1.08`) photo collage built by `buildCollage()` — 1/2/3-photo layouts (single full-bleed, 50/50 split, or hero+stacked-pair mosaic), pulled from up to 3 of the cluster's entries that have `mediaType === 'photo'`.
   - A circular selection badge overlaid top-right (checkmark when selected, translucent black when not).
   - Album name, formatted date range, entry count ("N items").
4. **Sticky footer** (position: absolute, bottom): only rendered once `selectedCount > 0`; shows "Add N Album(s)" button. A spacer view (`footerSpace`) is added to the scroll content so the footer never overlaps the last row.

## States
- `status`: `'loading' | 'empty' | 'results'` (line 265) — computed once on mount by calling `computeSuggestions(entries)` (from `TripAlbumSuggestionService`) against the entries snapshot at open time (explicitly *not* re-run if `entries` changes later — see the `eslint-disable-next-line react-hooks/exhaustive-deps` comment, line 291).
  - `loading`: `ActivityIndicator` + "Finding your trips…".
  - `empty`: shown both when clustering legitimately finds nothing AND when `computeSuggestions` throws (caught and treated identically, lines 283–284) — a real failure is indistinguishable from "no trips found" to the user.
  - `results`: the 2-column grid.
- **Per-card collage sub-states** inside `buildCollage()`: 0 photos → "No photos" placeholder tile; 1 photo → full-bleed; 2 → 50/50 split; 3+ → hero (67% width) + two stacked (33% width) — only ever samples up to 3 photos regardless of cluster size.
- **Selection**: `selectedIds: Set<string>`, toggled per-card tap; footer and subtitle only respond to `selectedCount > 0`.
- **Adding**: `isAdding` boolean disables the footer button and swaps its label to "Adding…"; on error, silently resets `isAdding` to `false` with no user-facing error message (`catch { setIsAdding(false); }`, lines 319–321) — a failed `createTripAlbum` call fails silently.
- **Country attribution**: each created album's `locationName` is only set if the cluster's detected `country !== 'Unknown'` (line 314).

## Data & context dependencies
- `useAppContext()`: `entries`, `createTripAlbum`, `refreshData` (called in a `finally` block after clustering, independent of success/failure — likely to pick up any backfilled metadata).
- `services/TripAlbumSuggestionService.computeSuggestions(entries)` — the actual clustering logic lives there, not in this screen; worth its own reference doc if the clustering heuristic (date/location grouping) needs documenting.
- `components/common/EntryPhotoImage` for collage tiles (handles fallback background color when an image fails to load).

## Interactions
- Tap-to-toggle selection (no long-press, no swipe/drag reordering).
- No pull-to-refresh — the suggestion list is a single point-in-time computation per screen visit.

## Theming notes
- Fully theme-driven (`useTheme().colors`); no light/dark-specific branching.
- Selection badge uses a fixed white border regardless of theme (`borderColor: 'white'`, line 191) — worth checking contrast in light mode where card backgrounds may already be light.

## Reference UI skeleton

```tsx
// source: src/screens/SuggestedAlbumsScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
return (
  <View style={styles.container}>
    <View style={styles.header}>
      <TouchableOpacity style={styles.backButton} onPress={() => navigation.goBack()}>
        <BackArrow width={19} height={21} color={colors.textPrimary} />
        <Text style={styles.backButtonText}>Back</Text>  {/* icon + text combined, unlike most other screens' bare icon */}
      </TouchableOpacity>
      <Text style={styles.title}>Suggested Trips</Text>
      {status === 'results' && (
        <Text style={styles.subtitle}>{suggestions.length} trips found · Tap to select</Text>
      )}
    </View>

    {status === 'loading' && (
      <View style={styles.centered}>
        <ActivityIndicator size="large" color={colors.textPrimary} />
        <Text style={styles.loadingText}>Finding your trips…</Text>
      </View>
    )}

    {status === 'empty' && (  // also covers a thrown clustering error — indistinguishable to the user, lines 282-284
      <View style={styles.centered}>
        <Text style={styles.emptyTitle}>No trip suggestions yet</Text>
        <Text style={styles.emptySubtext}>Add more experiences with location data{'\n'}to unlock trip suggestions.</Text>
      </View>
    )}

    {status === 'results' && (
      <ScrollView contentContainerStyle={styles.grid}>  {/* flexWrap row, justify space-between, 2 cols via cardWidth */}
        {suggestions.map((suggestion) => {
          const isSelected = selectedIds.has(suggestion.id);
          return (
            <TouchableOpacity key={suggestion.id} style={[styles.card, { width: cardWidth }]} onPress={() => toggleSelection(suggestion.id)}>
              <View style={styles.previewContainer}>  {/* aspectRatio: 1.08, borderRadius: 16, overflow hidden */}
                {/* buildCollage(suggestion.entries): 0 photos -> emptyPreview "No photos" tile
                    1 photo  -> full-bleed singlePreview
                    2 photos -> splitPreview: splitLeft (50%) + splitRight (50%)
                    3+ photos -> mosaicPreview: heroPreview (67% width) + stackedContainer (33%) with stackedTop/stackedBottom (50% height each) */}
                <View style={[styles.selectionBadge, isSelected ? styles.selectionBadgeSelected : styles.selectionBadgeUnselected]}>
                  {isSelected && <Text style={styles.checkmark}>✓</Text>}
                </View>
              </View>
              <View style={styles.albumInfo}>
                <Text style={styles.albumName} numberOfLines={1}>{suggestion.name}</Text>
                <Text style={styles.albumDate} numberOfLines={1}>{dateRange}</Text>
                <Text style={styles.albumCount}>{suggestion.entries.length} items</Text>
              </View>
            </TouchableOpacity>
          );
        })}
        {selectedCount > 0 && <View style={styles.footerSpace} />}  {/* height: 90 + insets.bottom, keeps footer from overlapping last row */}
      </ScrollView>
    )}

    {status === 'results' && selectedCount > 0 && (
      <View style={styles.footer}>  {/* position: absolute, bottom: 0, borderTopWidth hairline */}
        <TouchableOpacity style={[styles.addButton, isAdding && styles.addButtonDisabled]} onPress={handleAddAlbums} disabled={isAdding}>
          <Text style={styles.addButtonText}>{isAdding ? 'Adding…' : `Add ${selectedCount} Albums`}</Text>
        </TouchableOpacity>
      </View>
    )}
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.background },
  header: { paddingTop: insets.top + SPACING.sm, paddingHorizontal: SPACING.md, paddingBottom: SPACING.sm },
  title: { fontSize: 22, fontWeight: FONT_WEIGHTS.bold, color: colors.textPrimary, marginTop: SPACING.xs, marginBottom: SPACING.xs },
  subtitle: { fontSize: 14, color: colors.textSecondary, marginBottom: SPACING.sm },
  centered: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: SPACING.xl },
  grid: { padding: SPACING.md, flexDirection: 'row', flexWrap: 'wrap', justifyContent: 'space-between', rowGap: SPACING.md },
  previewContainer: { width: '100%', aspectRatio: 1.08, borderRadius: 16, backgroundColor: colors.surface, overflow: 'hidden' },
  singlePreview: { width: '100%', height: '100%' },
  splitPreview: { width: '100%', height: '100%', flexDirection: 'row' },
  splitLeft: { width: '50%', height: '100%' },
  splitRight: { width: '50%', height: '100%' },
  mosaicPreview: { width: '100%', height: '100%', flexDirection: 'row' },
  heroPreview: { width: '67%', height: '100%' },
  stackedContainer: { width: '33%', height: '100%', flexDirection: 'column' },
  stackedTop: { width: '100%', height: '50%' },
  stackedBottom: { width: '100%', height: '50%' },
  selectionBadge: { position: 'absolute', top: SPACING.sm, right: SPACING.sm, width: 26, height: 26, borderRadius: 13, borderWidth: 2, borderColor: 'white' },
  selectionBadgeSelected: { backgroundColor: colors.textPrimary },
  selectionBadgeUnselected: { backgroundColor: 'rgba(0,0,0,0.35)' },
  footerSpace: { height: 90 + insets.bottom },
  footer: { position: 'absolute', bottom: 0, left: 0, right: 0, backgroundColor: colors.background, borderTopWidth: StyleSheet.hairlineWidth, borderTopColor: colors.border, paddingHorizontal: SPACING.md, paddingTop: SPACING.md },
  addButton: { backgroundColor: colors.textPrimary, borderRadius: 14, paddingVertical: 14, alignItems: 'center' },
  addButtonDisabled: { opacity: 0.6 },
  addButtonText: { fontSize: 16, fontWeight: FONT_WEIGHTS.semibold, color: colors.background },
});
```

**Key technique(s):**
- The collage-layout branching (`buildCollage`, `src/screens/SuggestedAlbumsScreen.tsx:34-81`) is a good example of "3 or fewer, pick one of 4 fixed layouts" rather than a generic masonry algorithm — only ever samples up to 3 photos regardless of how many the cluster actually has.
- `cardWidth = (screenWidth - SPACING.md * 3) / 2` (line 271) is computed once per render from raw `Dimensions.get('window').width`, not `useWindowDimensions()` — won't react to an orientation change without a remount (moot on this iOS-only, portrait-locked app, but notable if that ever changes).

## Screenshot
_Not yet captured. TODO: capture light + dark for: loading, empty, results-with-1/2/3-photo collages, a selected card, and the footer with the "Add N Albums" CTA visible._

## Open questions / follow-ups
- Clustering failure and "genuinely no trips" both render as the same `empty` state (lines 282–284) — confirm with product whether a distinct error state is wanted, since right now a bug in `computeSuggestions` is invisible to the user and to whoever's triaging support reports.
- A failed `createTripAlbum` during `handleAddAlbums` (lines 319–321) has no visible error feedback (no `Alert`, unlike almost every other destructive/save action elsewhere in the app) — likely worth aligning with the `Alert.alert('Error', ...)` pattern used in `TripAlbumCreateScreen`/`TripAlbumDetailScreen`.
