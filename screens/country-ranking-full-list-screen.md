# Country Ranking Full List

**Route name:** `CountryRankingFullList`
**File:** `src/screens/CountryRankingFullListScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
The simplest of the four dashboard sub-pages: an un-truncated, scrollable version of the Review dashboard's "top countries by memory count" ranking. Pure display, no editing, no filtering. Serves the same stats-visualization need as `CountriesVisitedMapScreen` but for the ranked-list view of the same underlying data.

## Entry points & navigation
- **In:** only from `components/review/DashboardSection.tsx`, via `navigation.navigate('CountryRankingFullList', { rows: allCountryRanking })` — like the map screen, the ranking is computed once upstream in `DashboardSection` and just handed down as `rows`, not recomputed here.
- **Params:** `{ rows?: Array<{ country: string; memories: number }> }` (inferred from usage — `row.country`, `row.memories`).
- **Out:** back arrow only → `navigation.goBack()`. No row is tappable; this is a read-only list (unlike `CountriesVisitedMapScreen`, which lets you tap a country to change its category).

## Layout anatomy
1. Same fixed header pattern as its three sibling dashboard sub-pages: back arrow, centered title "Country ranking", spacer.
2. A single `ScrollView` of rows, each row: `"{index+1}. {flagEmoji} {countryName}"` on the left, `"{memories} memory|memories"` on the right, separated by a hairline bottom border.

## States
- **Empty** (`rows.length === 0`): centered "No countries in this list." text — the only conditional branch in the whole file.
- **Populated**: the ranked row list, in whatever order `rows` arrives in (ranking/sorting is `DashboardSection`'s responsibility, not this screen's).

## Data & context dependencies
- `useTheme()` → `colors`; `useSafeAreaInsets()`.
- `utils/countryUtils` → `getCountryFlagEmoji(countryName)` for the leading flag emoji per row.
- No `AppContext`/entries access at all — this screen is pure presentation over its `route.params`, making it the least stateful screen in this cluster.

## Interactions
- None beyond scrolling and the back button — no gestures, no animations.

## Theming notes
- Fully theme-aware (`colors.background`, `colors.border`, `colors.textPrimary/Secondary`), consistent with its siblings.

## Reference UI skeleton
This is the whole screen — nothing is stripped since there's no business logic beyond reading `route.params.rows`. Useful as a minimal-viable-subpage reference against the other 3 dashboard sub-pages.

```tsx
// source: src/screens/CountryRankingFullListScreen.tsx (full render tree — no business logic to strip)
function CountryRankingFullListScreen() {
  const rows = route.params?.rows ?? [];

  return (
    <View style={[styles.root /* flex:1 */, { backgroundColor: colors.background, paddingTop: insets.top }]}>
      <View style={[styles.header /* row, paddingHorizontal: SPACING.md, borderBottomWidth: hairline */, { borderBottomColor: colors.border }]}>
        <TouchableOpacity onPress={() => navigation.goBack()} style={styles.backBtn /* 40x40 */} accessibilityLabel="Back">
          <BackArrow width={22} height={22} color={colors.textPrimary} />
        </TouchableOpacity>
        <Text style={[styles.headerTitle /* 18 semibold, flex:1, centered */, { color: colors.textPrimary }]}>Country ranking</Text>
        <View style={styles.headerSpacer /* width:40 */} />
      </View>

      <ScrollView style={styles.scroll /* flex:1 */}
        contentContainerStyle={{ paddingHorizontal: SPACING.md, paddingTop: SPACING.md, paddingBottom: insets.bottom + SPACING.xl }}>
        {rows.length === 0 ? (
          <Text style={[styles.empty /* 14 */, { color: colors.textSecondary }]}>No countries in this list.</Text>
        ) : (
          rows.map((row, index) => (
            <View key={row.country} style={[styles.listRow /* row, justify:'space-between', borderBottomWidth: hairline, paddingVertical: SPACING.xs */, { borderBottomColor: colors.border }]}>
              <Text style={[styles.listLabel /* 14, flex:1 */, { color: colors.textPrimary }]}>
                {index + 1}. {getCountryFlagEmoji(row.country)} {row.country}
              </Text>
              <Text style={[styles.listMeta /* 13 */, { color: colors.textSecondary }]}>
                {row.memories} {row.memories === 1 ? 'memory' : 'memories'}
              </Text>
            </View>
          ))
        )}
      </ScrollView>
    </View>
  );
}
```

**Key technique(s):** none beyond the shared dashboard-subpage header shape (fixed back/title/spacer row) — see the same shape in `countries-visited-map-screen.md`, `full-country-calendar-screen.md`, and `day-detail-screen.md`.

## Screenshot
_Not yet captured. TODO: capture light + dark, populated list; empty state is low-value (rare/edge-case)._

## Open questions / follow-ups
- `key={row.country}` assumes country names are unique within `rows` — true for a ranking (one row per country) but worth confirming `DashboardSection` never produces duplicate country entries (e.g. from inconsistent casing/whitespace in captured country names) before treating this as safe.
- Same note as `CountriesVisitedMapScreen`: this screen's fixed-header/card layout is part of an undocumented shared "dashboard sub-page" convention across 4 screens — candidate for a `patterns/dashboard-subpages.md`.
