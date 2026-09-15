# Countries Visited Map

**Route name:** `CountriesVisitedMapFull`
**File:** `src/screens/CountriesVisitedMapScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
Full-screen, pinch-to-zoom-and-pan version of a world map card that lives on the Review dashboard, showing every country the user has visited. Also the *only* place a visited country's category (Home / Lived in / Visited / Layover) can be edited — the dashboard card is read-only display. Serves the Travel Sharer/Reflective personas' stats-and-visualization needs (see PRD Feature 7 / DR2).

## Entry points & navigation
- **In:** only from `components/review/DashboardSection.tsx`, via `navigation.navigate('CountriesVisitedMapFull', { visitedIsoCodes, countryNames })` — the visited-country list is computed once in `DashboardSection` and passed down rather than recomputed here.
- **Params:** `{ visitedIsoCodes?: string[]; countryNames?: string[] }`.
- **Out:** back arrow → `navigation.goBack()`. No other exits (no drill-down from a country row into e.g. `DayDetail` — categorizing is the only action).

## Layout anatomy
Deliberately mirrors every other full-screen dashboard sub-page (`DayDetail`, `FullCountryCalendar`, `CountryRankingFullList`) per the file's own header comment, "so nav chrome stays consistent":
1. Fixed header: back arrow, centered title "Countries visited", spacer.
2. Hero stat block: large country count + "country"/"countries" label, then `CategoryFilterChips` (toggleable filter row).
3. Map area (`flex: 1`, **not** inside the below `ScrollView`) — a bordered card (`colors.surface`/`colors.border`, same treatment as `DashboardCard`) containing `ZoomableWorldMap`.
4. Bounded-height (`maxHeight: 260`) `ScrollView` below the map, containing one more bordered card with the tappable `CountryCategoryList`.

## States
- **Filter active/inactive**: `filterCategory` toggles which category (if any) is highlighted on the map and used to filter the list; tapping an already-active chip clears it (toggle, not radio).
- **Category overrides loading**: `categoryOverrides` starts `{}` and is populated from `getCountryCategories()` (AsyncStorage-backed) on every screen focus via `useFocusEffect` — so edits made elsewhere (there's nowhere else, but structurally supports it) are picked up on return. Countries with no stored override default to `'visited'`.
- **Empty**: not explicitly handled — if `countryNames`/`visitedIsoCodes` are empty arrays (shouldn't normally happen since the dashboard card that links here only appears once countries exist), the hero stat shows "0 countries" and the list is empty; no dedicated empty-state copy.

## Data & context dependencies
- `useTheme()` → `colors`; `useSafeAreaInsets()`.
- `storage/countryCategoryStorage` → `getCountryCategories()` (read, on focus) and `setCountryCategory(isoCode, category)` (write, optimistic — local state updates immediately, storage write is fire-and-forget `void`).
- `components/review/ZoomableWorldMap`, `CountryCategoryList`, `CategoryFilterChips` — all three are this screen's real UI; the screen itself is mostly layout/state glue.

## Interactions
- Pinch-to-zoom-and-pan is implemented inside `ZoomableWorldMap` itself (not shown here) — this screen just gives it a fixed-height, **non-scrolling** container. The header comment is explicit about why: keeping the map's pan gesture *out* of a wrapping `ScrollView` avoids the two gesture recognizers fighting each other.
- Double-tap-to-reset is documented via `accessibilityLabel` on the map's wrapping card ("pinch to zoom and drag to pan, double-tap to reset") — the actual gesture logic lives in `ZoomableWorldMap`.

## Theming notes
- Fully theme-aware via `colors` from `useTheme()`; no hardcoded colors. `ZoomableWorldMap` and `CountryCategoryList` both receive `colors`/theme-derived props rather than reading theme themselves (prop-drilled, not each re-subscribing to `useTheme()`).

## Reference UI skeleton

```tsx
// source: src/screens/CountriesVisitedMapScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
function CountriesVisitedMapScreen() {
  // categoryOverrides refreshed from getCountryCategories() on every focus — :50-54
  // categories: visitedIsoCodes -> categoryOverrides[iso] ?? 'visited' — :56-62

  return (
    <View style={[styles.root /* flex:1 */, { backgroundColor: colors.background, paddingTop: insets.top }]}>
      <View style={[styles.header /* row, paddingHorizontal: SPACING.md, borderBottomWidth: hairline */, { borderBottomColor: colors.border }]}>
        <TouchableOpacity onPress={() => navigation.goBack()} style={styles.backBtn /* 40x40 */} accessibilityLabel="Back">
          <BackArrow width={22} height={22} color={colors.textPrimary} />
        </TouchableOpacity>
        <Text style={[styles.headerTitle /* 18 semibold, flex:1, centered */, { color: colors.textPrimary }]}>Countries visited</Text>
        <View style={styles.headerSpacer /* width:40, balances back button */} />
      </View>

      <View style={styles.topSection /* paddingHorizontal: SPACING.md */}>
        <Text style={[styles.heroStat /* 40, bold, lineHeight:44 */, { color: colors.textPrimary }]}>{countryNames.length}</Text>
        <Text style={[styles.heroLabel /* 14 */, { color: colors.textSecondary }]}>{countryNames.length === 1 ? 'country' : 'countries'}</Text>
        <CategoryFilterChips selected={filterCategory} onToggle={handleToggleFilter} />
      </View>

      {/* Map region deliberately sits OUTSIDE the ScrollView below — see Key technique */}
      <View style={styles.mapArea /* flex:1, paddingHorizontal: SPACING.md */}>
        <View style={[styles.mapCard /* flex:1, borderRadius:20, padding: SPACING.sm, overflow:'hidden' */, { backgroundColor: colors.surface, borderColor: colors.border }]}
          accessibilityLabel="Zoomable world map, pinch to zoom and drag to pan, double-tap to reset">
          <ZoomableWorldMap categories={categories} colors={colors} highlightCategory={filterCategory} />
        </View>
      </View>

      <ScrollView style={styles.listSection /* maxHeight: 260 */} contentContainerStyle={{ paddingBottom: insets.bottom + SPACING.xl }}>
        <View style={[styles.listCard /* borderRadius:20, padding: SPACING.md, marginHorizontal: SPACING.md */, { backgroundColor: colors.surface, borderColor: colors.border }]}>
          <Text style={[styles.listCardTitle /* 13 semibold */, { color: colors.textPrimary }]}>Tap a country to set its category</Text>
          <CountryCategoryList countryNames={countryNames} categories={categories} onChangeCategory={handleChangeCategory} filterCategory={filterCategory} />
        </View>
      </ScrollView>
    </View>
  );
}
```

**Key technique(s):**
- The map region (`styles.mapArea`, :101-108) is deliberately kept *outside* the `ScrollView` below it — its own header comment (:8-15) explains why: the map's pan gesture would otherwise fight the wrapping `ScrollView`'s pan responder. This "fixed header → hero stat → non-scrolling map card → bounded scroll region" shape is shared verbatim by `DayDetail`/`FullCountryCalendar`/`CountryRankingFullList` (see Open questions).
- `ZoomableWorldMap`/`CountryCategoryList` receive `colors` as a prop rather than each independently subscribing to `useTheme()` — prop-drilled theming, unlike most other screens in this app.

## Screenshot
_Not yet captured. TODO: capture light + dark, with a filter chip active (map highlighting one category) and inactive._

## Open questions / follow-ups
- The screen's own header comment explicitly documents its relationship to 3 sibling screens (`DayDetail`, `FullCountryCalendar`, `CountryRankingFullList`) as sharing one "full-screen dashboard sub-page" layout convention (fixed header, card-based sections). That's a real, named pattern worth promoting into `patterns/` (e.g. `patterns/dashboard-subpages.md`) rather than leaving it discoverable only by reading four separate screen files' comments.
