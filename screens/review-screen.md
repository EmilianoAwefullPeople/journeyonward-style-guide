# Review Screen

**Route name:** `Review` (bottom tab, part of `MainTabs`)
**File:** `src/screens/ReviewScreen.tsx`
**Type:** Tab screen

## Purpose
The second of the app's two bottom tabs (Home, Review). Houses three sub-sections — Stats ("dashboard"), Your Trips ("tripAlbums"), and Gallery ("bookmarks") — giving the user aggregate/reflective views over everything they've captured, as opposed to Home's live-capture-first feed.

## Entry points & navigation
- Tab bar → "Review" (always mounted alongside "Home" per `AppNavigator.tsx`'s `Tab.Navigator`, `lazy: true`, `unmountOnBlur: false`).
- Route param: `route.params?.initialSection` (typed `ReviewSectionParam`), normalized via `normalizeSection()` against `VALID_REVIEW_SECTIONS = ['stats', 'tripAlbums', 'bookmarks']`, defaulting to `'stats'` if absent/invalid. Used e.g. by `TripAlbumCreateScreen`'s post-save reset, which passes `{ initialSection: 'tripAlbums' }` so a newly created album is immediately visible.
- Settings gear icon (top-right of the segment row) → `navigation.navigate('Settings')`.
- Entry/journal/postcard/album taps → `EntryDetail` / `JournalDetail` / `PostcardEditor` / `TripAlbumDetail` respectively, each with the relevant id param.
- "Create album" → `TripAlbumCreate`; "Suggest trips" → `SuggestedAlbums`; per-album Recap → `JourneyRecapPlayer`; per-album Slideshow → `MemoriesSlideshow` — all delegated from within `TripAlbumsSection`'s callbacks, wired here.
- Re-tapping the *already-active* "Review" tab (`tabPress` navigation event, lines 77–93) scrolls the currently active section back to top — same top-tap-to-scroll-up convention as `TripAlbumDetailScreen`.

## Layout anatomy
1. `MeshGradientBackground` full-bleed.
2. Header block: settings-gear row (right-aligned circular button) → `SegmentControl` (from `components/review/SegmentControl`, 3 segments: Stats / Your Trips / Gallery) → conditionally, a "This Trip ✓" filter pill (only shown when `travelModeEnabled && normalizedSection === 'stats'` — i.e. only on the Stats segment, since Gallery has its own inline filter and Trip Albums always shows everything).
3. Body: all three sections mounted simultaneously, switched by visibility rather than unmount — see States for why Gallery is handled differently from the other two.

## States
- **Section**: `activeSection`/`normalizedSection` — `'stats' | 'tripAlbums' | 'bookmarks'`. Note: PRD/screenshot naming calls this "Gallery" and "Your Trips" in the UI, but the internal state values are `'bookmarks'` and `'tripAlbums'` — worth knowing when grepping the codebase for either the PRD's or the UI's terminology.
- **Trip filter ("This Trip")**: `tripFilterActive`, defaults to `tripFilterAvailable = travelModeEnabled && activeTripAlbums.length > 0` (i.e. auto-on when travel mode is active and there's a current trip), but user-toggleable; a `useEffect` re-syncs it whenever `tripFilterAvailable` changes externally (e.g. travel mode toggled off in Settings mid-session, lines 54–56). When active, `displayEntries`/`displayJournals` are filtered down to just the active album(s)' items (lines 63–71); Trip Albums section is unaffected by this filter (always shows all albums).
- **Gallery panel mounting technique (distinct from the other two)**: Gallery/`BookmarksSection` sits in an `absoluteFillObject` layer, always mounted, toggled only via `pointerEvents`/accessibility props — never `display: none` — specifically so its internal `FlashList` always has valid layout dimensions and never needs to re-measure when switching tabs (see comment, lines 169–174). It's simply painted over by the Stats/TripAlbums panels (which carry opaque backgrounds) when those are active.
- **Stats/TripAlbums panels**: toggled via `display: 'flex' | 'none'` with an explicit `backgroundColor: colors.background` specifically to occlude the Gallery layer beneath them (comment, lines 196–199) — the opposite technique from Gallery, and the reason there's no slide/fade transition between segments here (unlike `TripAlbumDetailScreen`'s animated section switch).
- **`DashboardSection`'s own `tripFilterActive` prop**: passed as `tripFilterActive && activeTripAlbums.length > 0` (line 221) — a slightly different guard than the screen-level `tripFilterAvailable`, worth noting if the dashboard's filtered/unfiltered states ever seem inconsistent with the pill's visible state.

## Data & context dependencies
- `useAppContext()`: `entries`, `journals`, `tripAlbums`, `travelModeEnabled`, `activeTripAlbums`, `postcards`.
- `navigation/mainTabBarVisibility.useMainTabBarVisibility()`: `visibilityProgress`, `reportScrollOffset` — shared hide-on-scroll tab bar behavior, passed down into each section so their internal scroll views can drive it.
- Delegates all actual list/grid/filter rendering to three components under `src/components/review/`: `SegmentControl`, `BookmarksSection`, `TripAlbumsSection`, `DashboardSection` — this screen is essentially a router/state-holder, not where the visual "grid" or "filter chips" logic lives.

## Interactions
- Segment switching is instant (no animation) by design (see the display:none/absoluteFill technique above) — contrast with `TripAlbumDetailScreen`'s fade+translate transition on its structurally similar segment control.
- Trip-filter pill: simple opacity dim (`opacity: 0.5`) when inactive, full opacity + checkmark suffix when active.

## Theming notes
- Fully `useTheme().colors`-driven; no hardcoded light/dark branches in this file itself.

## Reference UI skeleton

```tsx
// source: src/screens/ReviewScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
return (
  <View style={[styles.container, { backgroundColor: colors.background }]}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

    <View style={[styles.segmentContainer, { paddingTop: insets.top + 12 }]}>
      <View style={styles.settingsRow}>
        <TouchableOpacity style={[styles.settingsButton, { backgroundColor: colors.surfaceVariant, borderColor: colors.border }]} onPress={() => navigation.navigate('Settings')}>
          <Ionicons name="settings-outline" size={22} color={colors.textPrimary} />
        </TouchableOpacity>
      </View>
      <View style={styles.segmentRow}>
        <SegmentControl activeSegment={normalizedSection} onSegmentChange={setActiveSection} />  {/* Stats / Your Trips / Gallery */}
      </View>
      {travelModeEnabled && normalizedSection === 'stats' && (  // only shown on Stats — Gallery has its own inline filter, Trip Albums always shows all
        <View style={styles.tripFilterRow}>
          <TouchableOpacity style={[styles.tripFilterButton, { backgroundColor: colors.surface }, !tripFilterActive && { opacity: 0.5 }]} onPress={() => setTripFilterActive(v => !v)}>
            <Text style={styles.tripFilterButtonText}>{tripFilterActive ? 'This Trip ✓' : 'This Trip'}</Text>
          </TouchableOpacity>
        </View>
      )}
    </View>

    <View style={styles.contentContainer}>
      {/* Gallery: always mounted, absolute-fill, occlusion via pointerEvents/accessibility only —
          never display:none, so its internal FlashList never needs to re-measure on tab switch. */}
      <View
        style={StyleSheet.absoluteFillObject}
        pointerEvents={normalizedSection === 'bookmarks' ? 'auto' : 'none'}
        accessibilityElementsHidden={normalizedSection !== 'bookmarks'}
      >
        <BookmarksSection
          entries={displayEntries} journals={displayJournals} postcards={postcards}
          onEntryPress={handleEntryPress} onJournalPress={handleJournalPress} onPostcardPress={handlePostcardPress}
          scrollRef={bookmarksScrollRef} onScrollReport={reportScrollOffset} tabBarVisibilityProgress={visibilityProgress}
          tripFilterAvailable={tripFilterAvailable} tripFilterActive={tripFilterActive} onTripFilterToggle={() => setTripFilterActive(v => !v)}
        />
      </View>

      {/* Stats / TripAlbums: opposite technique — display:none when inactive, opaque background
          to cover the Gallery layer painted beneath. No slide/fade transition either way. */}
      <View style={[styles.sectionPanel, { display: normalizedSection === 'tripAlbums' ? 'flex' : 'none', backgroundColor: colors.background }]}>
        <TripAlbumsSection
          tripAlbums={tripAlbums} activeTripAlbums={activeTripAlbums} entries={entries} journals={journals}
          onAlbumPress={handleAlbumPress} onCreateAlbumPress={handleCreateAlbumPress} onSuggestTripsPress={handleSuggestTripsPress}
          onRecapPress={handleRecapPress} onSlideshowPress={handleSlideshowPress} scrollRef={tripAlbumsScrollRef}
        />
      </View>
      <View style={[styles.sectionPanel, { display: normalizedSection === 'stats' ? 'flex' : 'none', backgroundColor: colors.background }]}>
        <DashboardSection
          entries={displayEntries} journals={displayJournals} tripAlbums={tripAlbums}
          tripFilterActive={tripFilterActive && activeTripAlbums.length > 0}
          isVisible={normalizedSection === 'stats'} scrollRef={statsScrollRef}
        />
      </View>
    </View>
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1 },
  segmentContainer: { paddingTop: 50, paddingBottom: SPACING.sm, backgroundColor: 'transparent' },
  settingsRow: { flexDirection: 'row', justifyContent: 'flex-end', paddingHorizontal: SPACING.md, marginBottom: SPACING.xs },
  segmentRow: { paddingHorizontal: SPACING.md },
  settingsButton: { width: 40, height: 40, borderRadius: 20, borderWidth: 1, alignItems: 'center', justifyContent: 'center' },
  contentContainer: { flex: 1 },
  sectionPanel: { flex: 1 },
  tripFilterRow: { flexDirection: 'row', paddingHorizontal: SPACING.md, paddingTop: SPACING.xs, paddingBottom: SPACING.xs },
  tripFilterButton: { paddingHorizontal: SPACING.md, paddingVertical: SPACING.sm, borderRadius: 20 },
  tripFilterButtonText: { fontSize: 14, fontWeight: FONT_WEIGHTS.semibold },
});
```

**Key technique(s):**
- The two opposite occlusion techniques for the three panels (`src/screens/ReviewScreen.tsx:175-227`) are worth carrying forward as a documented pattern given how easy it'd be to "fix" one to match the other without realizing they're intentionally different — Gallery avoids `FlashList` remeasure by never unmounting; Stats/TripAlbums avoid needing an opacity/z-index dance by just painting over Gallery with an opaque background.
- `TripAlbumDetailScreen` reuses this same 3-segment idea but with the *opposite* choice (fade+translate transition, all panels via `display:none` including its "All" equivalent) — the inconsistency is intentional per that screen's own comments, not a bug, but worth knowing both exist if unifying segment-switch UX later.

## Screenshot
_Already partially covered: `screens/review-grid-filters.png` (Gallery/grid + filter chips, light) and `screens/review-trips-dark.png` (Your Trips segment, dark) exist per `screens/README.md`. Still needed: the Stats/Dashboard segment in both themes, and the "This Trip" filter pill in both its on and off states._

## Open questions / follow-ups
- **PRD mismatch**: `PRD_JourneyOn_MVP.md` Flow 7 describes Review as a **Grid/Map view toggle** with trip/time/location/draft filters. The actual implementation is a 3-way segment control (Stats / Your Trips / Gallery) with no top-level Map view and no time-range/location/draft filters at this screen's level (Map only exists one level down, inside `TripAlbumDetailScreen`). Whatever filtering exists lives inside `BookmarksSection`'s own filter chips (per `screens/review-grid-filters.png`'s "Experiences / Bookmarks / Photos / Notes" chips) rather than at the `ReviewScreen` level described in the PRD. Flag for product/design to reconcile intent vs. build.
- `SegmentControl`, `BookmarksSection`, `TripAlbumsSection`, and `DashboardSection` (all in `src/components/review/`) carry the actual UI weight of this screen and deserve their own component-level docs/screenshots rather than being fully described secondhand here or in `trip-album-detail-screen.md` (which reuses `DashboardSection`/`MapViewSection`). Recommend a `patterns/review-sections.md` doc as a follow-up so this doc and `trip-album-detail-screen.md` can both point to one source of truth instead of each re-describing the same components.
