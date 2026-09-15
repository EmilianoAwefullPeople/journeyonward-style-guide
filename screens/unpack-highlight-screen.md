# Unpack Highlight Screen

**Route name:** `UnpackHighlight`
**File:** `src/screens/unpack/UnpackHighlightScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
Stage 3a of 4: the review step where the user decides which clustered photo-moments to keep as real `Entry` records, and stars a minimum number of them as "highlights" (bookmarked). This is also the point where the `TripAlbum` is finally written to the database — deferred from Stage 1 so an abandoned flow never leaves an empty album behind.

## Entry points & navigation
- From `UnpackProcessingScreen`'s reset (Path A) → arrives with `{ albumId, suggestions, albumName, albumStartDate, albumEndDate, albumDescription }`.
- From `UnpackTripDetailsScreen` (Path B, step-4 save) → same param shape.
- On "Continue" (enabled once enough highlights are starred) → persists the album + accepted entries, sets `isBookmarked` on highlighted entries, then `navigate('UnpackUnpacking', { albumId, highlightedIds })`.
- Back chevron → `navigation.goBack()`.
- Empty-suggestions edge case exposes its own "← Go back" button in the empty state (in addition to the header back chevron).

## Layout anatomy
1. `MeshGradientBackground` backdrop.
2. Header: back chevron + `UnpackStepBar` (`currentStage={3}`).
3. `FlatList` of `SuggestionCard`s, with a `ListHeaderComponent` containing: screen title "Highlight what mattered", a subtitle stating the moment count and the required-highlight count, and (when >1 suggestion) a "Keep all" button.
4. Each `SuggestionCard`: up to 3 photo thumbnails side-by-side, a gradient scrim with date (+ photo count if multiple), a 3-color gradient accent strip along the top edge when highlighted, and an action row (Skip / star-toggle / Keep-toggle).
5. Sticky footer CTA, one of three states (see States).
6. Empty state (replaces the list): "No moments found" + explanatory body + "← Go back" button.

## States
- **Empty**: `suggestions.length === 0` → dedicated empty-state layout (see above), no footer.
- **Populated, not enough highlights**: footer shows a disabled button with a star-outline icon and "Star N more to continue" (singular/plural aware).
- **Populated, enough highlights, idle**: footer shows an enabled "Continue →" button.
- **Saving**: `saving` → footer swaps to a spinner + "Saving…", both `Continue` interactions blocked.
- Per-card state: each suggestion independently tracks `isAccepted` (in `acceptedIds`) and `isHighlighted` (in `highlightedIds`) — starring a card auto-accepts it; un-accepting a card auto-un-highlights it (see excerpt).
- `minHighlights = Math.min(3, suggestions.length)` — the required-highlight threshold adapts down if fewer than 3 suggestions exist at all (so a 1- or 2-photo trip isn't stuck unable to reach "3").
- Post-save highlight-flagging is itself a small async state machine: `pendingHighlights` is set after `persistSelectedExperiences` resolves, and a `useEffect` waits for `entries` (from context) to refresh and contain matching timestamps before calling `updateEntry({ isBookmarked: true })` on each and only then navigating onward — because the newly created entries don't exist in `entries` until a context refresh completes.

## Data & context dependencies
- `useAppContext()` → `entries`, `tripAlbums`, `createEntry`, `updateEntry`, `refreshData`, `createTripAlbum`.
- `persistSelectedExperiences` (`src/services/ExperienceImportService`) — converts accepted suggestions into real `Entry` records.
- `createTripAlbum(album, 'unpack')` — the `'unpack'` second argument presumably tags the album's creation source/origin.

## Interactions
- Skip: removes a suggestion from the list entirely (and from both id sets).
- Star (highlight) toggle: adding a highlight also force-adds it to `acceptedIds`; removing a highlight leaves accept-state untouched.
- Keep (accept) toggle: un-accepting also force-removes the highlight, since a skipped/un-accepted item can't remain "starred."
- "Keep all": bulk-sets `acceptedIds` to every current suggestion id (does not affect highlights).
- "Continue": see the highlight-flagging state machine above.

## Theming notes
- Card border/accent uses a fixed brand color (`COLORS.royalGold`) for the highlighted state regardless of theme, plus a fixed 3-stop `FLOW_GRADIENT` (`babyBlue → royalGold → fern`) reused across all three Unpack review/tagging screens as a visual "you're in this flow" signal.
- Everything else (surface, border, text) is theme-aware via `useTheme()`.

## Reference UI skeleton
Whole render tree, business logic stripped. `FLOW_GRADIENT = [COLORS.babyBlue, COLORS.royalGold, COLORS.fern]` (reused across all three Unpack review/tagging screens as a "you're in this flow" signal). `PHOTO_HEIGHT = 180`, `MAX_THUMBS = 3`.

```tsx
// source: src/screens/unpack/UnpackHighlightScreen.tsx (skeleton — business logic stripped)

// ── SuggestionCard sub-component ────────────────────────────────────────────
const SuggestionCard = ({ suggestion, isAccepted, isHighlighted, onSkip, onToggleAccept, onToggleHighlight }) => (
  <View style={[cardStyles.container, { borderColor: isHighlighted ? COLORS.royalGold : colors.border, borderWidth: isHighlighted ? 1.5 : hairline }]}>
    {isHighlighted && (
      <LinearGradient colors={FLOW_GRADIENT} start={{x:0,y:.5}} end={{x:1,y:.5}} style={cardStyles.highlightStrip} /> // 3px accent strip, top edge
    )}
    <View style={cardStyles.photoArea}> {/* height: PHOTO_HEIGHT */}
      {thumbs.length > 0 ? (
        <View style={cardStyles.thumbRow}>{thumbs.map((uri, i) => <Image key={i} source={{ uri }} style={cardStyles.thumb} />)}</View>
      ) : (
        <View style={[cardStyles.thumbRow, cardStyles.noPhoto]}><Ionicons name="image-outline" size={32} /></View>
      )}
      <LinearGradient colors={['transparent', 'rgba(0,0,0,0.6)']} style={cardStyles.scrim}>
        <AppText style={cardStyles.photoDate}>{formattedDate}{suggestion.assets.length > 1 && ` · ${count} photos`}</AppText>
      </LinearGradient>
    </View>
    <View style={cardStyles.actions}> {/* row, border-top, paddingHorizontal: SPACING.md */}
      <TouchableOpacity onPress={onSkip}><AppText style={cardStyles.skipText}>Skip</AppText></TouchableOpacity>
      <View style={cardStyles.rightActions}> {/* flex:1, justify-end, row, gap: SPACING.sm */}
        <TouchableOpacity
          onPress={onToggleHighlight}
          style={[cardStyles.starBtn, { backgroundColor: isHighlighted ? COLORS.royalGold+'22' : colors.surfaceVariant, borderColor: isHighlighted ? COLORS.royalGold : colors.border }]}
        >
          <Ionicons name={isHighlighted ? 'star' : 'star-outline'} size={17} color={isHighlighted ? COLORS.royalGold : colors.textSecondary} />
        </TouchableOpacity>
        <TouchableOpacity
          onPress={onToggleAccept}
          style={[cardStyles.keepBtn, { backgroundColor: isAccepted ? COLORS.babyBlue : colors.surfaceVariant }]}
        >
          <Ionicons name={isAccepted ? 'checkmark' : 'add'} size={14} />
          <AppText style={cardStyles.keepText}>{isAccepted ? 'Kept' : 'Keep'}</AppText>
        </TouchableOpacity>
      </View>
    </View>
  </View>
);

// ── Screen ───────────────────────────────────────────────────────────────────
const ListHeader = (
  <View style={styles.listHeader}> {/* paddingTop: SPACING.md, paddingBottom: SPACING.lg */}
    <AppText style={styles.screenTitle}>Highlight what mattered</AppText> {/* Playfair Display italic bold, 26px */}
    <AppText style={styles.screenSubtitle}>{count} moment{s} found. Star at least {minHighlights} to continue.</AppText>
    {suggestions.length > 1 && (
      <TouchableOpacity style={styles.selectAllBtn} onPress={handleSelectAll}><AppText style={styles.selectAllText}>Keep all</AppText></TouchableOpacity>
    )}
  </View>
);

return (
  <View style={[styles.container, { backgroundColor: colors.background }]}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
    <View style={[styles.header, { paddingTop: insets.top + SPACING.sm }]}>
      <View style={styles.headerRow}>
        <TouchableOpacity style={styles.backBtn} onPress={() => navigation.goBack()}><Ionicons name="chevron-back" size={22} /></TouchableOpacity>
      </View>
      <UnpackStepBar currentStage={3} />
    </View>

    {isEmpty ? (
      <View style={styles.emptyState}> {/* flex:1, center, gap: SPACING.md */}
        <AppText style={styles.emptyTitle}>No moments found</AppText>
        <AppText style={styles.emptyBody}>Try going back and selecting different photos.</AppText>
        <TouchableOpacity style={styles.emptyBackBtn} onPress={() => navigation.goBack()}><AppText style={styles.emptyBackText}>← Go back</AppText></TouchableOpacity>
      </View>
    ) : (
      <FlatList
        data={suggestions}
        keyExtractor={item => item.id}
        ListHeaderComponent={ListHeader}
        contentContainerStyle={[styles.listContent, { paddingBottom: insets.bottom + 120 }]}
        renderItem={({ item }) => (
          <SuggestionCard
            suggestion={item} isAccepted={acceptedIds.has(item.id)} isHighlighted={highlightedIds.has(item.id)}
            onSkip={() => handleSkip(item.id)} onToggleAccept={() => handleToggleAccept(item.id)} onToggleHighlight={() => handleToggleHighlight(item.id)}
          />
        )}
      />
    )}

    {!isEmpty && (
      <View style={styles.footer}> {/* absolute bottom, border-top */}
        {saving ? (
          <View style={[styles.ctaBtn, { opacity: 0.8 }]}><ActivityIndicator size="small" /><AppText style={styles.ctaText}>Saving…</AppText></View>
        ) : canContinue ? (
          <TouchableOpacity style={styles.ctaBtn} onPress={handleContinue}><AppText style={styles.ctaText}>Continue →</AppText></TouchableOpacity>
        ) : (
          <TouchableOpacity style={[styles.ctaBtn, { opacity: 0.6 }]} disabled>
            <Ionicons name="star-outline" size={16} />
            <AppText style={styles.ctaText}>{needed === 1 ? 'Star 1 more to continue' : `Star ${needed} more to continue`}</AppText>
          </TouchableOpacity>
        )}
      </View>
    )}
  </View>
);
```

**Key technique(s):** The star/accept coupling is the one piece of business logic not obvious from the UI alone: un-accepting a card force-removes its highlight (a skipped/un-accepted item can't stay "starred"), and starring a card force-adds it to accepted (you can't highlight something you haven't kept) — `file:325-351`. Post-save, `pendingHighlights` triggers a `useEffect` that waits for `entries` (from context) to refresh and contain matching timestamps before calling `updateEntry({ isBookmarked: true })`, because the newly created entries don't exist in `entries` until a context refresh completes — `file:291-317`.

## Screenshot
_Not yet captured. TODO: capture light + dark for: populated list with a mix of highlighted/accepted/plain cards, the disabled "Star N more" footer, the enabled "Continue" footer, the saving footer, and the empty state._

## Open questions / follow-ups
- The post-save `useEffect` matches newly-created entries back to suggestions **by exact `timestamp` string equality** (`entries.filter(e => timestamps.includes(e.timestamp))`) rather than by an id returned from `persistSelectedExperiences` — this is fragile if two accepted suggestions ever share an identical timestamp, or if `persistSelectedExperiences` ever normalizes/rewrites timestamps during entry creation. Worth confirming this can't collide.
