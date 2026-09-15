# Journey Recap Candidates

**Route name:** `JourneyRecapCandidates`
**File:** `src/screens/JourneyRecapCandidatesScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
A picker screen: lists every trip album that has enough data to generate a Journey Recap slideshow, so the user can tap one and open the recap player. Serves the Reflective/Travel Sharer personas at the moment they want to revisit a trip.

## Entry points & navigation
- **In:** No live call site navigates here — `grep`ing the whole `src/` tree for `navigate('JourneyRecapCandidates'` returns nothing. It is registered in `AppNavigator.tsx` and typed in `navigation/types.ts` (`JourneyRecapCandidates: undefined`), but every other recap entry point (Review dashboard, Home, `TripAlbumDetailScreen`, `StudioJourneyRecapRow`, the Unpack flow) now navigates straight to `JourneyRecapPlayer` with a known `albumId`, bypassing this list. See "Open questions" below.
- **Params:** none (`undefined` in `RootStackParamList`).
- **Out:** tapping a candidate card → `navigation.navigate('JourneyRecapPlayer', { albumId: c.id })`. Empty state's CTA → `navigation.navigate('TripAlbumCreate')`. Back arrow / error-state "Back" → `navigation.goBack()`.

## Layout anatomy
1. `MeshGradientBackground` full-bleed behind everything (`patterns/effects-and-atmosphere.md`).
2. Header: back arrow (`BackArrow` icon) + "Journey Recap" title, `paddingTop: insets.top`.
3. Body — one of four mutually exclusive renders (see States): loading spinner, error message + retry, empty-state CTA, or a `ScrollView` of candidate cards.
4. Each candidate card: 72×72 two-up thumbnail strip (up to 2 preview images side by side, or a blank surface if none), title + date range, trailing chevron. 2pt colored border (`colors.borderLight`) — same "card" treatment used elsewhere (see `patterns/cards-and-feed-grid.md`).

## States
- **`loading`** (initial): centered `LoadingSpinner` + "Finding your trips…". Triggered on mount and on every `loadCandidates()` retry.
- **`error`**: shown if `getRecapCandidates()` throws. Alert-circle icon, error text, gradient "Retry" button (`COLORS.babyBlue → royalGold → fern` linear gradient — same gradient used for the tab bar/capture button per `patterns/navigation-and-tab-bar.md`), and a plain-text "Back".
- **`empty`**: shown when `getRecapCandidates()` returns `[]`. Images-outline icon, "No trips to recap yet", same gradient CTA but routed to trip-album creation instead of retry.
- **`list`**: the populated `ScrollView` of cards, described above.

## Data & context dependencies
- `useAppContext()` → `entries`, `tripAlbums` (raw), fed into `getRecapCandidates(entries, tripAlbums)` from `utils/recapCandidates` to derive the `RecapCandidate[]` (id, displayName, dateRange, previewImageUris).
- `useTheme()` → `colors`, `isDark` (passed to `MeshGradientBackground` as `lightMode={!isDark}`).
- `useSafeAreaInsets()` for header/scroll padding.
- `logger` (`utils/logger`) for warn/error logging around navigation and candidate-loading failures.

## Interactions
- Plain `TouchableOpacity` taps throughout (`activeOpacity={0.7}`/`0.8`), no gesture-handler or Reanimated usage on this screen.
- All navigation calls are wrapped in `try/catch` with `logger.warn` on failure — a defensive pattern repeated across this screen (back, create-album, select-candidate all guarded individually).

## Theming notes
- Fully theme-aware: every color reference goes through `colors` from `useTheme()` (`colors.background`, `colors.textPrimary/Secondary`, `colors.surfaceVariant`, `colors.borderLight`) — no hardcoded hex except the shared brand gradient (`COLORS.babyBlue/royalGold/fern`), which is intentionally static across themes.
- `MeshGradientBackground` is told explicitly which mode to render (`lightMode={!isDark}`) rather than inferring it itself.

## Reference UI skeleton

```tsx
// source: src/screens/JourneyRecapCandidatesScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
function JourneyRecapCandidatesScreen() {
  const [state, setState] = useState<'loading' | 'list' | 'empty' | 'error'>('loading');
  // loadCandidates() populates `candidates`/`errorMessage` and sets state — :173-190

  return (
    <View style={styles.container /* flex:1, backgroundColor: colors.background */}>
      <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

      <View style={styles.header /* row, paddingTop: insets.top + SPACING.sm, paddingBottom: SPACING.md */}>
        <TouchableOpacity onPress={handleBack} style={styles.backButton} accessibilityLabel="Go back">
          <BackArrow color={colors.textPrimary} width={24} height={24} />
        </TouchableOpacity>
        <Text style={styles.headerTitle /* 18, semibold */}>Journey Recap</Text>
      </View>

      {state === 'loading' && (
        <View style={styles.centerContent /* flex:1, centered, paddingHorizontal: SPACING.xl */}>
          <LoadingSpinner size="large" />
          <Text style={styles.loadingText /* 16, textSecondary, marginTop: SPACING.lg */}>Finding your trips…</Text>
        </View>
      )}

      {state === 'error' && (
        <View style={styles.centerContent}>
          <Ionicons name="alert-circle-outline" size={48} color={colors.textSecondary} />
          <Text style={styles.errorText}>{errorMessage}</Text>
          <TouchableOpacity style={styles.primaryButton /* paddingV: SPACING.md, borderRadius:12, overflow:'hidden' */} onPress={loadCandidates}>
            <LinearGradient
              colors={[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]}
              start={{ x: 0, y: 0 }} end={{ x: 1, y: 0 }}
              style={[StyleSheet.absoluteFillObject, { borderRadius: 12 }]}
            />
            <Text style={styles.primaryButtonText}>Retry</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.secondaryButton} onPress={handleBack}>
            <Text style={styles.secondaryButtonText}>Back</Text>
          </TouchableOpacity>
        </View>
      )}

      {state === 'empty' && (
        <View style={styles.centerContent}>
          <Ionicons name="images-outline" size={48} color={colors.textSecondary} />
          <Text style={styles.emptyTitle}>No trips to recap yet</Text>
          <Text style={styles.emptySubtext}>Add a trip album and experiences to get your first recap.</Text>
          <TouchableOpacity style={styles.primaryButton} onPress={handleCreateAlbum /* -> TripAlbumCreate */}>
            {/* same gradient fill as error's Retry button */}
            <Text style={styles.primaryButtonText}>Create a trip album</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.secondaryButton} onPress={handleBack}>
            <Text style={styles.secondaryButtonText}>Back</Text>
          </TouchableOpacity>
        </View>
      )}

      {state === 'list' && (
        <ScrollView style={styles.scroll /* flex:1 */} contentContainerStyle={styles.scrollContent /* paddingHorizontal: SPACING.md */}>
          {candidates.map((c) => (
            <TouchableOpacity
              key={c.id}
              style={[styles.card, { borderColor: colors.borderLight /* borderWidth:2, borderRadius:12, marginBottom: SPACING.md */ }]}
              onPress={() => handleSelectCandidate(c) /* -> JourneyRecapPlayer, { albumId: c.id } */}
            >
              <View style={[styles.cardInner, { backgroundColor: colors.surfaceVariant } /* row, padding: SPACING.md, alignItems:'center' */]}>
                <View style={styles.cardThumbnails /* 72x72, borderRadius:8, row, overflow:'hidden' */}>
                  {c.previewImageUris.length > 0 ? (
                    c.previewImageUris.slice(0, 2).map((uri, i) => (
                      <Image key={i} source={{ uri }} style={styles.cardThumb /* flex:1 */} resizeMode="cover" />
                    ))
                  ) : (
                    <View style={[styles.cardThumb, { backgroundColor: colors.surface }]} />
                  )}
                </View>
                <View style={styles.cardTextWrap /* flex:1, marginLeft: SPACING.md */}>
                  <Text style={styles.cardTitle /* 16, semibold */} numberOfLines={2}>{c.displayName}</Text>
                  <Text style={styles.cardDate /* 13, textSecondary */} numberOfLines={1}>{c.dateRange}</Text>
                </View>
                <Ionicons name="chevron-forward" size={22} color={colors.textSecondary} />
              </View>
            </TouchableOpacity>
          ))}
        </ScrollView>
      )}
    </View>
  );
}
```

**Key technique(s):**
- The state machine (`loading | list | empty | error`) is driven by one `useCallback` (`loadCandidates`, :173-186) that's reused as the retry handler — no separate "retry" code path to keep in sync.
- All four states share the identical header (back arrow + "Journey Recap" title); only the body below it swaps, which is why the skeleton renders the header once outside the state branches.

## Screenshot
_Not yet captured. TODO: capture light + dark for `list` (populated) and `empty` states; `loading`/`error` are low-value to screenshot (transient/rare)._

## Open questions / follow-ups
- **This screen appears to be dead/orphaned in the current navigation graph.** It's fully built (all 4 states implemented, error handling, empty-state CTA) but nothing currently links to it — every other recap entry point now passes `albumId` directly to `JourneyRecapPlayer`. Worth asking the team: is this an intentional "recap hub" slated for a future entry point (e.g. a dedicated "Recaps" tab/section), leftover from before per-album entry points existed, or safe to delete? Don't silently remove it without confirming — it may be the intended landing spot for a not-yet-wired-up nav affordance.
- If it is meant to come back, note it duplicates logic already present in `getRecapCandidates` + `Review`'s "Your Trips" carousel (`review-trips-dark.png` in this same `screens/` folder) — worth checking those don't drift apart if both surface "recappable trips."
