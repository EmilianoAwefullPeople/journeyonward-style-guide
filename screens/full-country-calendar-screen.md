# Full Country Calendar

**Route name:** `FullCountryCalendar`
**File:** `src/screens/FullCountryCalendarScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
Full-screen, scrollable month-by-month calendar (not trip-filtered — all memories across the user's whole history) showing which country each day was spent in, using a "flag per day" visualization. Also where the user can select a *range* of days and assign a country to all of them at once, for days with no automatically-detected country. Supports the Retrospective Traveler's "consolidate and make sense of everything" need and the general country-tracking feature seen across this whole cluster.

## Entry points & navigation
- **In:** only from `components/review/DashboardSection.tsx`'s calendar card, via `navigation.navigate('FullCountryCalendar')` (no params — this screen computes its own date range and data, unlike its map/ranking siblings which receive precomputed data).
- **Params:** none.
- **Out:** tapping a day → `navigation.navigate('DayDetail', { dayKey })`. Back arrow → `navigation.goBack()`.

## Layout anatomy
1. Fixed header (same 4-screen dashboard-subpage convention as its siblings): back arrow, centered "Calendar" title, spacer.
2. A single `ScrollView` (ref-tracked) containing one `DashboardDayFlagCalendar` per month in range, each wrapped in a `View` with `onLayout` used to auto-scroll to the current month on first render.
3. `CountryRangeAssignSheet` — a bottom-sheet-style modal, shown only while a day-range selection is in progress.

## States
- **Initial month-range resolution** (async, on mount): `getOrInitCalendarJoinMonth(entries, journals)` determines the earliest month to show (the user's "join month" — presumably first-ever entry/journal, computed once and cached), combined with `getLatestMemoryYearMonth()` to compute the full `[start, end]` range via `computeFullCalendarStartMonth`. Until this resolves, `monthRange` is `[]` and nothing renders in the scroll area.
- **Country-by-coordinate resolution** (async, separate effect, capped at `FULL_CALENDAR_MAX_GEOCODE_COORDS = 120` distinct rounded coordinates): reverse-geocodes any entry/journal coordinates not already known from a seeded `country` field, soft-failing (`catch { /* soft-fail */ }`) per-coordinate rather than aborting the whole batch.
- **Range-select mode** (`rangeSelectDayKeys !== null`): triggered by `DashboardDayFlagCalendar`'s `onRangeSelectionEnd` callback (a drag-select gesture living inside that component, not this screen) once 2+ days are selected; shows `CountryRangeAssignSheet` to pick one country for the whole range, then persists via `setDayCountrySnapshotsBatch` and refreshes `calendarSnapshots`.
- **Refresh on return from `DayDetail`**: `useFocusEffect` re-fetches snapshots for all visible day-keys, in case a country was just recorded on the `DayDetail` screen for a previously-unknown day.
- **Auto-scroll-to-current-month**: a one-shot ref guard (`scrolledToCurrentRef`) fires on the *first* month `onLayout` call matching `currentMonthKey`, so it doesn't re-trigger on every layout pass.

## Data & context dependencies
- `useAppContext()` → `entries`, `journals` (both needed — days can have either or both).
- `storage/calendarJoinMonthStorage`: `computeFullCalendarStartMonth`, `currentYearMonth`, `enumerateMonthsInclusive`, `getLatestMemoryYearMonth`, `getOrInitCalendarJoinMonth`.
- `storage/dayCountrySnapshotStorage`: `getDayCountrySnapshotsForDayKeys`, `setDayCountrySnapshotsBatch` — the persisted, manually-set country-per-day overrides (distinct from a country auto-derived from an entry's own `country`/coordinates).
- `utils/dateUtils` → `getCountryFromCoordinates` (reverse geocoding).
- `utils/dayKeyUtils` → `localDayKeyFromDate`, `localDayKeysInMonth`.
- `utils/calendarDayCountryMap` → `buildCaptureCountryByDayKey`, `buildMergedDayCountryDisplay` — merges auto-derived and manually-set country data into one per-day display value, with (implicitly, by argument order) manual snapshots taking precedence over derived data.
- `components/review/DashboardDayFlagCalendar` (the actual month-grid renderer) and `CountryRangeAssignSheet`.

## Interactions
- Drag-to-select a day range — implemented inside `DashboardDayFlagCalendar`, surfaced to this screen only as a completed `dayKeys[]` callback; this screen doesn't itself handle any pan/gesture.
- Scroll-to-month-on-mount via `ScrollView.scrollTo({ y, animated: false })`, computed from each month `View`'s measured layout `y`.

## Theming notes
- Fully theme-aware; passes individual theme tokens (`colors.textPrimary/Secondary/Tertiary`, `colors.surfaceVariant`) down as separate props to `DashboardDayFlagCalendar` rather than a single `colors` object — that component's prop signature is more granular than its siblings' (`ZoomableWorldMap`/`CountryCategoryList` take a whole `colors` object).

## Reference UI skeleton

```tsx
// source: src/screens/FullCountryCalendarScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
function FullCountryCalendarScreen() {
  // monthRange resolved async: getOrInitCalendarJoinMonth + getLatestMemoryYearMonth -> [start,end] — :67-84
  // countryByCoord: reverse-geocodes up to FULL_CALENDAR_MAX_GEOCODE_COORDS=120 distinct coords, soft-fails per-coord — :86-116
  // captureCountryByDay = buildCaptureCountryByDayKey(entries, journals, countryByCoord) — :118-121
  // rangeSelectDayKeys: set by DashboardDayFlagCalendar's onRangeSelectionEnd once 2+ days selected — :140-155

  return (
    <View style={[styles.root /* flex:1 */, { backgroundColor: colors.background, paddingTop: insets.top }]}>
      <View style={[styles.header /* row, borderBottomWidth: hairline */, { borderBottomColor: colors.border }]}>
        <TouchableOpacity onPress={() => navigation.goBack()} style={styles.backBtn /* 40x40 */} accessibilityLabel="Back">
          <BackArrow width={22} height={22} color={colors.textPrimary} />
        </TouchableOpacity>
        <Text style={[styles.headerTitle /* 18 semibold, flex:1, centered */, { color: colors.textPrimary }]}>Calendar</Text>
        <View style={styles.headerSpacer /* width:40 */} />
      </View>

      <ScrollView ref={scrollRef} style={styles.scroll /* flex:1 */}
        contentContainerStyle={{ paddingHorizontal: SPACING.md, paddingTop: SPACING.md, paddingBottom: insets.bottom + SPACING.xl }}>
        {monthRange.map((ym) => {
          const merged = buildMergedDayCountryDisplay(/* keys, captureCountryByDay, calendarSnapshots */);
          return (
            // Collapsed: one representative month — every entry in monthRange renders the same structure
            <View key={`${ym.year}-${ym.monthIndex0}`} style={{ marginBottom: SPACING.lg }} onLayout={handleMonthLayout /* auto-scrolls to current month once, :157-167 */}>
              <DashboardDayFlagCalendar
                width={innerWidth} year={ym.year} monthIndex0={ym.monthIndex0}
                displayByDay={merged}
                textPrimary={colors.textPrimary} textSecondary={colors.textSecondary} textTertiary={colors.textTertiary}
                todayDayKey={todayDayKey} todayPillBackground={colors.surfaceVariant} todayDayNumberColor={colors.textPrimary}
                onDayPress={handleDayPress /* -> DayDetail, { dayKey } */}
                onRangeSelectionEnd={handleRangeSelectionEnd}
              />
            </View>
          );
        })}
      </ScrollView>

      <CountryRangeAssignSheet
        visible={rangeSelectDayKeys != null}
        dayCount={rangeSelectDayKeys?.length ?? 0}
        onCancel={() => setRangeSelectDayKeys(null)}
        onSave={handleRangeSelectSave}
      />
    </View>
  );
}
```

**Key technique(s):**
- `DashboardDayFlagCalendar` owns the actual month-grid rendering and the drag-to-select gesture; this screen only ever sees a completed `dayKeys[]` callback (`onRangeSelectionEnd`) — no pan/gesture code lives in this file itself.
- The range-select persistence flow (`handleRangeSelectSave`, :144-155) writes a batch snapshot then re-fetches *all* visible day-keys' snapshots (not just the changed range) to refresh the calendar in one pass.
- Shares the same fixed-header/card layout convention as `countries-visited-map-screen.md`, `country-ranking-full-list-screen.md`, and `day-detail-screen.md` — see this screen's own header comment (:8-15).

## Screenshot
_Not yet captured. TODO: capture light + dark of the populated month scroll, and the `CountryRangeAssignSheet` open mid-range-selection._

## Open questions / follow-ups
- `FULL_CALENDAR_MAX_GEOCODE_COORDS = 120` is a hard cap on how many distinct rounded coordinates get reverse-geocoded per screen visit — for a very prolific user (120+ distinct 2-decimal-precision locations across their whole history) some days would permanently show no auto-derived country unless a manual snapshot is set. Worth flagging as a known scaling limit, not a bug, but one a designer/PM should know about.
- Coordinate rounding to 2 decimal places (`toFixed(2)`, ~1.1km precision) is used both here and independently reimplemented in `CountriesVisitedMapScreen`'s sibling logic path (`seededCountryByCoord` in this file uses the same `toFixed(2)` keying) — worth a shared `roundCoordKey()` util if not already centralized, to avoid the two call sites drifting to different precisions later.
- Same dashboard-subpage layout note as its three siblings — candidate content for a `patterns/dashboard-subpages.md`.
