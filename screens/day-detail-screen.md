# Day Detail

**Route name:** `DayDetail`
**File:** `src/screens/DayDetailScreen.tsx`
**Type:** Stack screen, **`presentation: 'modal'`** (the only one of this cluster's 4 dashboard sub-pages presented as a modal rather than a card)

## Purpose
Shows every memory (entries + journals) captured on one specific calendar day, lets the user page horizontally between adjacent days without leaving the screen, and — only when no country is already known for that day — lets them record one. The single-day counterpart to `FullCountryCalendarScreen`'s month grid.

## Entry points & navigation
- **In:** `FullCountryCalendarScreen` (tapping a day) and `components/review/DashboardSection.tsx` (a separate day-tap entry point on the dashboard itself, not just from the full calendar) — both via `navigation.navigate('DayDetail', { dayKey })`.
- **Params:** `{ dayKey: string }` (a `YYYY-M-D`-shaped local-day key, per `parseDayKey`).
- **Out:** close (top-left `BackArrow`) → `navigation.goBack()`. Tapping a memory card → `navigation.navigate('EntryDetail', { entryId })` or `navigation.navigate('JournalDetail', { journalId })`. Prev/next day chevrons and the swipe gesture **do not navigate** — they update `activeDayKey` in place, keeping the same screen instance mounted.

## Layout anatomy
1. Fixed header: close button, center cluster (prev-day chevron, date label + optional country subtitle, next-day chevron), spacer.
2. Conditional country-prompt block (see States) directly under the header.
3. A `GestureDetector`-wrapped `ScrollView`: `PinterestGrid` of the day's photo/media entries (if any), followed by a list of journal cards (title + up to 6 lines of content + time), or a centered "No memories on this date." + "Swipe to browse other days." empty state.
4. Whole screen wrapped in `KeyboardAvoidingView` (`behavior: 'padding'` on iOS) since the country-input `AppTextInput` can bring up the keyboard.

## States
- **`resolvingCountry`**: true while the day's country is being resolved (see Data dependencies); gates whether the country-input prompt can show at all (`showCountryInput = !resolvingCountry && !country`), so there's no flash-of-input before resolution completes.
- **`showCountryInput`**: shown only once resolution finishes and still finds no country — a labeled `AppTextInput` with live filtered suggestions (`ALL_COUNTRY_NAMES`, capped at `MAX_SUGGESTIONS = 6`) rendered as tappable chips below it.
- **`saving`**: disables suggestion chips while `setDayCountrySnapshot` is in flight.
- **`hasMemories`**: `dayEntries.length > 0 || dayJournals.length > 0` — toggles between the populated grid/journal-list layout and the empty-state message.
- **Day navigation**: `activeDayKey` drives every derived value (`dateLabel`, `dayEntries`, `dayJournals`, country resolution) via `useMemo`/`useEffect` dependencies — changing it is the *only* state transition needed to "page" to a different day; no remount, no navigation stack change.

## Data & context dependencies
- `useAppContext()` → `entries`, `journals`, filtered client-side by `dayKeyFromTimestampIso(...) === activeDayKey`.
- Country resolution is a 3-tier fallback, re-run on every `activeDayKey` change: (1) the most recent entry's own persisted `country` field, reversed-and-found so the *latest* entry of the day wins; (2) if none, reverse-geocode the most recent item with coordinates via `getCountryFromCoordinates`; (3) if that also fails/is absent, fall back to a previously manually-set `getDayCountrySnapshot(activeDayKey)`.
- `storage/dayCountrySnapshotStorage` → `getDayCountrySnapshot`, `setDayCountrySnapshot` (the single-day counterpart to `FullCountryCalendarScreen`'s batch variant).
- `utils/countryUtils` → `ALL_COUNTRY_NAMES`, `getCountryFlagEmoji`.
- `components/review/PinterestGrid` (same masonry grid documented in `patterns/cards-and-feed-grid.md`) for the day's photo entries.

## Interactions
- **Swipe-to-change-day**: a `Gesture.Pan()` from `react-native-gesture-handler`, deliberately scoped with `.activeOffsetX([-20, 20]).failOffsetY([-14, 14])` so it only claims clearly-horizontal drags — vertical scrolling of the memories list underneath is left alone. `.runOnJS(true)` is used so the commit logic runs as plain JS-thread state updates rather than a Reanimated worklet; the in-file comment cross-references `PostcardFramedPhotoSlot.tsx` as the sibling case for that same worklet-avoidance pattern. Commit threshold: `SWIPE_COMMIT_DISTANCE = 60`pt.
- Tapping the header's prev/next chevrons is a discrete alternative to swiping, calling the same `goToPrevDay`/`goToNextDay` callbacks.

## Theming notes
- Fully theme-aware (`colors.background/border/textPrimary/textSecondary/textTertiary/surface`), consistent with its dashboard-subpage siblings. Journal card text uses `FONTS.playfairDisplay` (italic) — the one screen in this cluster that pulls in the serif/journal-specific type family instead of `FONTS.inter`, worth noting for typography consistency checks.

## Reference UI skeleton

```tsx
// source: src/screens/DayDetailScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
function DayDetailScreen() {
  // activeDayKey starts at route.params.dayKey; swipeGesture/chevrons shift it in place — :63, :69-87
  // country resolved 3-tier: persisted entry.country -> reverse-geocode -> manual snapshot — :114-155
  const showCountryInput = !resolvingCountry && !country;

  return (
    <KeyboardAvoidingView style={[styles.root /* flex:1 */, { backgroundColor: colors.background }]} behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
      <View style={[styles.header /* row, paddingTop: insets.top + SPACING.sm, borderBottomWidth: hairline */, { borderBottomColor: colors.border }]}>
        <TouchableOpacity onPress={() => navigation.goBack()} style={styles.backBtn /* 40x40 */} accessibilityLabel="Close">
          <BackArrow width={22} height={22} color={colors.textPrimary} />
        </TouchableOpacity>

        <View style={styles.headerCenter /* flex:1, row, centered */}>
          <TouchableOpacity onPress={goToPrevDay} style={styles.chevronBtn /* 28x28 */} accessibilityLabel="Previous day">
            <Ionicons name="chevron-back" size={20} color={colors.textSecondary} />
          </TouchableOpacity>
          <View style={styles.headerTitleWrap /* flex:1, centered, minWidth:0 */}>
            <Text style={[styles.headerTitle /* 15 semibold */, { color: colors.textPrimary }]} numberOfLines={1}>{dateLabel}</Text>
            {country ? (
              <Text style={[styles.headerSubtitle /* 13 */, { color: colors.textSecondary }]}>{getCountryFlagEmoji(country)} {country}</Text>
            ) : null}
          </View>
          <TouchableOpacity onPress={goToNextDay} style={styles.chevronBtn} accessibilityLabel="Next day">
            <Ionicons name="chevron-forward" size={20} color={colors.textSecondary} />
          </TouchableOpacity>
        </View>

        <View style={styles.headerSpacer /* width:40 */} />
      </View>

      {showCountryInput ? (
        <View style={[styles.countryPrompt /* paddingHorizontal: SPACING.md, paddingVertical: SPACING.md, borderBottomWidth: hairline */, { borderBottomColor: colors.border }]}>
          <Text style={[styles.countryPromptLabel /* 15 semibold */, { color: colors.textPrimary }]}>Which country were you in?</Text>
          <AppTextInput value={countryInput} onChangeText={setCountryInput} placeholder="Type a country name"
            style={[styles.countryInput /* 15, borderWidth: hairline, borderRadius:10 */, { color: colors.textPrimary, borderColor: colors.border, backgroundColor: colors.surface }]}
            onSubmitEditing={() => handleSaveCountry(countryInput)} />
          {suggestions.length > 0 ? (
            <View style={styles.suggestionsWrap /* row, wrap, gap: SPACING.xs */}>
              {suggestions.map((name) => (
                <TouchableOpacity key={name} style={[styles.suggestionChip /* borderRadius:16, borderWidth: hairline */, { borderColor: colors.border, backgroundColor: colors.surface }]}
                  onPress={() => handleSaveCountry(name)} disabled={saving}>
                  <Text style={[styles.suggestionText /* 13 */, { color: colors.textPrimary }]}>{getCountryFlagEmoji(name)} {name}</Text>
                </TouchableOpacity>
              ))}
            </View>
          ) : null}
        </View>
      ) : null}

      <GestureDetector gesture={swipeGesture /* Gesture.Pan(), see Key technique */}>
        <View style={styles.swipeArea /* flex:1 */}>
          <ScrollView keyboardShouldPersistTaps="handled" contentContainerStyle={{ paddingTop: SPACING.md, paddingBottom: insets.bottom + SPACING.xl, flexGrow: 1 }}>
            {hasMemories ? (
              <>
                {dayEntries.length > 0 ? (
                  <View style={styles.gridWrap /* paddingHorizontal: SPACING.md, centered */}>
                    <PinterestGrid entries={dayEntries} onEntryPress={handleEntryPress} />
                  </View>
                ) : null}
                {dayJournals.length > 0 ? (
                  <View style={styles.journalsWrap /* paddingHorizontal: SPACING.md, gap: SPACING.md */}>
                    {/* Collapsed: one representative journal card — every dayJournals entry renders the same structure */}
                    <TouchableOpacity style={[styles.journalCard /* borderRadius:16, padding: SPACING.md */, { backgroundColor: colors.surface }]}
                      onPress={() => handleJournalPress(journal)} accessibilityLabel="Journal entry">
                      {journal.title ? <Text style={[styles.journalTitle /* fontFamily: FONTS.playfairDisplay, semibold */, { color: colors.textPrimary }]} numberOfLines={1}>{journal.title}</Text> : null}
                      <Text style={[styles.journalText /* fontFamily: FONTS.playfairDisplay, italic, lineHeight:22 */, { color: colors.textPrimary }]} numberOfLines={6}>{journal.content}</Text>
                      <Text style={[styles.journalTime /* 11 */, { color: colors.textTertiary }]}>{/* formatted time */}</Text>
                    </TouchableOpacity>
                  </View>
                ) : null}
              </>
            ) : (
              <View style={styles.emptyState /* flex:1, centered, paddingVertical: SPACING.xxl */}>
                <Text style={[styles.emptyText /* 15 */, { color: colors.textSecondary }]}>No memories on this date.</Text>
                <Text style={[styles.emptyHint /* 13 */, { color: colors.textTertiary }]}>Swipe to browse other days.</Text>
              </View>
            )}
          </ScrollView>
        </View>
      </GestureDetector>
    </KeyboardAvoidingView>
  );
}
```

**Key technique(s):**
- The scoped swipe gesture (`swipeGesture`, :76-87): `Gesture.Pan().activeOffsetX([-20,20]).failOffsetY([-14,14])` claims only clearly-horizontal drags, so it coexists with the vertical `ScrollView` underneath without a gesture conflict; `.runOnJS(true)` deliberately avoids a Reanimated worklet (the in-file comment cross-references `PostcardFramedPhotoSlot.tsx` as the sibling case).
- Day-to-day paging never navigates or remounts — changing `activeDayKey` alone drives every derived value (`dateLabel`, `dayEntries`, `dayJournals`, country resolution) via `useMemo`/`useEffect`.
- The only screen in this 4-screen dashboard-subpage family presented as a `modal` rather than a `card` (see Open questions) — otherwise shares the same fixed back/title/spacer header shape as `countries-visited-map-screen.md`, `country-ranking-full-list-screen.md`, and `full-country-calendar-screen.md`.

## Screenshot
_Not yet captured. TODO: capture light + dark for: populated day (grid + journal cards), empty day, and the country-input prompt with suggestion chips showing._

## Open questions / follow-ups
- This is the only one of the 4 dashboard-subpage screens presented as a `modal` rather than a `card` (see `AppNavigator.tsx:816-824`) — worth confirming that's an intentional distinction (e.g. "day detail is a quick peek, the other three are drill-downs") rather than an inconsistency, since visually the header/back-button pattern otherwise matches its siblings exactly.
- The 3-tier country-resolution fallback (persisted entry country → reverse-geocode → manual snapshot) closely parallels logic in `FullCountryCalendarScreen` (`buildCaptureCountryByDayKey`/`buildMergedDayCountryDisplay`) but is implemented as its own inline `useEffect` here rather than calling into those same calendar utils — two independent implementations of a very similar "what country was this day" resolution. Worth checking they can't disagree (e.g. this screen shows country X for a day, the calendar month grid shows country Y for the same day) before v2.0 touches either.
- Same dashboard-subpage layout-convention note as its 3 siblings.
