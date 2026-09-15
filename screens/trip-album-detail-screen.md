# Trip Album Detail Screen

**Route name:** `TripAlbumDetail`
**File:** `src/screens/TripAlbumDetailScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
The single-trip view: shows one album's name/location/date range plus three sub-views (All Memories, Dashboard, Map) of just that album's entries and journals. Reuses the same segmented-control pattern as the global Review screen but scoped to one album.

## Entry points & navigation
- Reached from Review's Trip Albums grid/list (tap an album card) — route param `{ albumId: string }` (required, `route.params.albumId`).
- **Edit**: 3-dots menu → "Edit" → `navigation.navigate('TripAlbumEdit', { albumId })` (renders `TripAlbumCreateScreen` in edit mode — see `trip-album-create-screen.md`).
- **Delete**: 3-dots menu → "Delete" → confirmation `Alert` → `deleteTripAlbum(albumId)` → `navigation.goBack()`.
- **Recap**: segment control's recap action → `navigation.navigate('JourneyRecapPlayer', { albumId })` — only available when `tripFinished` is true (see States).
- Tapping an entry/journal card inside any sub-section navigates to `EntryDetail`/`JournalDetail` with the respective id.
- Back arrow → `navigation.goBack()`.

## Layout anatomy
1. `MeshGradientBackground` full-bleed.
2. Header: back arrow, album name (up to 2 lines), optional location line (only if `album.locationName` is set), formatted date range, and an absolutely-positioned "..." menu button top-right.
3. `TripAlbumSegmentControl` (from `src/components/review/`) — three segments: All (`'all'`), Dashboard (`'dashboard'`), Map (`'map'`), plus a conditional Recap action.
4. Section body — one of three panels, all mounted simultaneously and toggled via `display: 'flex'/'none'` (not unmounted) so scroll position and animated values persist per section:
   - **All** → `BookmarksSection` (entries + journals list/grid for this album).
   - **Dashboard** → `DashboardSection` with `tripFilterActive` forced true and `tripAlbums={[album]}` (i.e. the same dashboard component used elsewhere, pre-scoped to just this one album).
   - **Map** → `MapViewSection` for this album's entries/journals.

## States
- **Error — album not found**: if `tripAlbums.find(a => a.id === albumId)` returns nothing, renders an early-return error layout (back arrow + "Album not found" / "This album may have been deleted") instead of the normal screen (lines 294–314). This can happen if the album was deleted in another session/tab while this screen was still mounted.
- **Recap eligibility**: `tripFinished = isTripFinished(album)` (via `utils/albumUtils`) gates whether the segment control's Recap button (`showRecap`) is shown at all — recap is only offered once the trip's date range has passed.
- **Location line conditional**: `album.locationName` is only rendered if present — no fallback placeholder text.
- **3-dots menu open/closed/exiting**: `menuVisible` / `menuExiting` drive a fade+scale-in dropdown (`Animated.parallel` on `menuOverlayOpacity` + `menuDropdownScale`); closing animates out before actually unmounting (`closeMenuAnimated`, lines 229–252) so a menu action (Edit/Delete) only fires after the close animation completes via its callback.
- **Reduce Motion**: `useReduceMotion()` hook zeroes both the section-switch fade/slide durations (`durationOut`/`durationIn` become 0) — the section transition animation is fully skipped rather than just shortened.
- **Section switch animation**: every time `activeSection` changes, content resets to `opacity: 0, translateY: -10` then animates back in (lines 205–227) — applies even to the currently-hidden panels' shared `Animated.View` wrapper.
- **Delete confirmation**: native two-button `Alert` with explicit destructive styling and copy clarifying entries survive deletion but lose their album association.

## Data & context dependencies
- `useAppContext()`: `tripAlbums`, `entries`, `journals`, `deleteTripAlbum`.
- `useTheme()`: `colors`, `isDark`.
- `useSafeAreaInsets()`, `useWindowDimensions()` — for the scale-factor responsive math (see Reference code excerpt).
- `useReduceMotion()` custom hook (`src/hooks/useReduceMotion.ts`) — accessibility setting.
- `utils/albumUtils.isTripFinished(album)`.

## Interactions
- 3-dots menu: `Animated.timing` fade (`Easing.out/in(Easing.cubic)`) + scale (`Easing.bezier(0.16, 1, 0.3, 1)` on open, cubic on close) — a spring-like custom bezier rather than `Animated.spring`.
- Section content transition: opacity + `translateY` `Animated.parallel`, respecting Reduce Motion.
- All three section panels are always mounted (visibility via `display: none`, plus `accessibilityElementsHidden`/`importantForAccessibility` toggled) rather than conditionally rendered — preserves each section's scroll position (`allScrollRef`/`dashboardScrollRef`/`mapScrollRef`) when switching tabs, and tapping the *already-active* segment scrolls that section back to top (`scrollActiveSectionToTop`, wired via `onActiveSegmentPress`).

## Theming notes
- No hardcoded light/dark branching apart from the delete-menu-item red (`#DC2626`, not theme-driven — same literal used in the delete `Alert`'s destructive semantics).
- All layout numbers are scaled by `scaleFactor = screenWidth / 390` (390px = reference iPhone width per `constants.ts`) rather than using the shared `SPACING`/`DIMENSIONS` scale directly for header positioning — see excerpt.

## Reference UI skeleton

```tsx
// source: src/screens/TripAlbumDetailScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)

// Error state — album not found (early return, no header content/segment control)
if (!album) {
  return (
    <View style={styles.container}>
      <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
      <TouchableOpacity style={[styles.backButton, { top: topBarTop, left: 25 * scaleFactor }]} onPress={handleBack}>
        <BackArrow width={19 * scaleFactor} height={21 * scaleFactor} color={colors.textPrimary} />
      </TouchableOpacity>
      <View style={styles.errorContainer}>
        <Text style={styles.errorText}>Album not found</Text>
        <Text style={styles.errorSubtext}>This album may have been deleted</Text>
      </View>
    </View>
  );
}

return (
  <View style={styles.container}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

    {/* Header */}
    <View style={[styles.header, { paddingTop: topBarTop }]}>
      <TouchableOpacity style={styles.backButton} onPress={handleBack}>
        <BackArrow width={19 * scaleFactor} height={21 * scaleFactor} color={colors.textPrimary} />
      </TouchableOpacity>
      <View style={styles.headerContent}>
        <Text style={[styles.albumName, { fontSize: 24 * scaleFactor }]} numberOfLines={2}>{album.name}</Text>
        {album.locationName && (  // no fallback placeholder — line omitted entirely if absent
          <Text style={[styles.albumLocation, { fontSize: 15 * scaleFactor }]} numberOfLines={1}>{album.locationName}</Text>
        )}
        <Text style={[styles.albumDate, { fontSize: 14 * scaleFactor }]}>{dateRange}</Text>
      </View>
    </View>

    {/* 3-dots menu button, absolutely positioned top-right */}
    <TouchableOpacity
      style={[styles.menuButton, { top: topBarTop, right: 25 * scaleFactor }]}
      onPress={() => (menuVisible || menuExiting) ? closeMenuAnimated() : setMenuVisible(true)}
    >
      <Text style={styles.menuButtonText}>...</Text>
    </TouchableOpacity>

    {(menuVisible || menuExiting) && (
      <Animated.View style={[styles.menuOverlay, { opacity: menuOverlayOpacity }]} pointerEvents="box-none">
        <TouchableWithoutFeedback onPress={() => closeMenuAnimated()}>
          <View style={StyleSheet.absoluteFill} />
        </TouchableWithoutFeedback>
        <Animated.View style={[styles.menuDropdown, { top: topBarTop + 40, right: 25 * scaleFactor, transform: [{ scale: menuDropdownScale }] }]}>
          <TouchableOpacity style={styles.menuOption} onPress={handleEdit}>
            <Ionicons name="settings" size={20} color={colors.textPrimary} />
            <Text style={styles.menuOptionText}>Edit</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.menuOption} onPress={handleDelete}>
            <Ionicons name="trash-outline" size={20} color="#DC2626" />
            <Text style={[styles.menuOptionText, styles.menuOptionTextDanger]}>Delete</Text>
          </TouchableOpacity>
        </Animated.View>
      </Animated.View>
    )}

    <View style={styles.segmentContainer}>
      <TripAlbumSegmentControl
        activeSegment={activeSection}
        onSegmentChange={setActiveSection}
        onActiveSegmentPress={scrollActiveSectionToTop}
        onRecapPress={handleRecapPress}
        showRecap={tripFinished}
      />
    </View>

    {/* All 3 panels always mounted; opacity+translateY Animated.View wraps whichever is display:flex */}
    <Animated.View style={[styles.content, { opacity: sectionOpacity, transform: [{ translateY: sectionTranslateY }] }]}>
      <View style={[styles.sectionPanel, { display: activeSection === 'all' ? 'flex' : 'none' }]}
            accessibilityElementsHidden={activeSection !== 'all'}>
        <BookmarksSection entries={albumEntries} journals={albumJournals} onEntryPress={handleEntryPress} onJournalPress={handleJournalPress} scrollRef={allScrollRef} />
      </View>
      <View style={[styles.sectionPanel, { display: activeSection === 'dashboard' ? 'flex' : 'none' }]}
            accessibilityElementsHidden={activeSection !== 'dashboard'}>
        <DashboardSection entries={albumEntries} journals={albumJournals} tripAlbums={[album]} tripFilterActive isVisible={activeSection === 'dashboard'} scrollRef={dashboardScrollRef} />
      </View>
      <View style={[styles.sectionPanel, { display: activeSection === 'map' ? 'flex' : 'none' }]}
            accessibilityElementsHidden={activeSection !== 'map'}>
        <MapViewSection entries={albumEntries} journals={albumJournals} onEntryPress={handleEntryPress} onJournalPress={handleJournalPress} scrollRef={mapScrollRef} />
      </View>
    </Animated.View>
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.background },
  header: { paddingTop: 50, paddingHorizontal: SPACING.md, paddingBottom: SPACING.sm, flexDirection: 'row', alignItems: 'flex-start', position: 'relative' },
  backButton: { marginRight: SPACING.md, marginTop: 4 },
  menuButton: { position: 'absolute', top: 50, right: SPACING.md, paddingVertical: SPACING.xs, paddingHorizontal: SPACING.sm, zIndex: 100 },
  menuOverlay: { ...StyleSheet.absoluteFillObject, zIndex: 200, backgroundColor: 'rgba(0,0,0,0.3)' },
  menuDropdown: { position: 'absolute', backgroundColor: colors.surface, borderRadius: 8, minWidth: 120, paddingVertical: 4, elevation: 8 },
  menuOption: { flexDirection: 'row', alignItems: 'center', paddingVertical: 12, paddingHorizontal: 16, gap: 10 },
  menuOptionTextDanger: { color: '#DC2626' },
  segmentContainer: { paddingHorizontal: SPACING.md, paddingVertical: SPACING.sm },
  albumName: { ...TEXT_STYLES.headerGreeting, fontSize: 24, marginBottom: SPACING.xs, color: colors.textPrimary },
  albumLocation: { ...TEXT_STYLES.entryInput, fontSize: 15, color: colors.textSecondary, marginBottom: SPACING.xs },
  albumDate: { ...TEXT_STYLES.entryInput, fontSize: 14, color: colors.textSecondary },
  content: { flex: 1 },
  errorContainer: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: SPACING.xxl },
  sectionPanel: { flex: 1 },
});
```

**Key technique(s):**
- Custom per-screen responsive scaling for header positioning (`src/screens/TripAlbumDetailScreen.tsx:53-56`), independent of the app's shared `SPACING`/`DIMENSIONS` tokens — worth flagging since other screens generally lean on those tokens alone:
```tsx
const { width: screenWidth } = useWindowDimensions();
const scaleFactor = screenWidth / 390;
const topBarTop = insets.top + (107.07 - 74 + 20) * scaleFactor;
```
- All 3 section panels are permanently mounted (`display: none` + accessibility props, not conditional rendering) so each panel's own `ScrollView` keeps its scroll position across tab switches (`src/screens/TripAlbumDetailScreen.tsx:396-435`).

## Screenshot
_Not yet captured. TODO: capture light + dark for: All/Dashboard/Map segments, the 3-dots menu open, the "album not found" error state, and the Recap-eligible vs. not-eligible segment control._

## Open questions / follow-ups
- The "album not found" error path (lines 294–314) renders a different, simpler layout (no segment control, no header content) — confirm this is an intentional degraded state and not just an unfinished error screen; worth a dedicated screenshot since it's easy to trigger only via race conditions (hard to hit by just clicking through the app normally).
- `DashboardSection` and `MapViewSection` are shared with the global Review screen (see `review-screen.md`) — their detailed behavior/spec should live in one place (probably `patterns/`) rather than being re-described per screen; flagging here so the Review-screen doc and this one don't duplicate/diverge.
