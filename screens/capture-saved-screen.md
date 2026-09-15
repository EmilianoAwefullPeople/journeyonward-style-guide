# Capture Saved

**Route name:** `CaptureSaved`
**File:** `src/screens/CaptureSavedScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `gestureEnabled: false`)

## Purpose
Post-save confirmation screen for a just-captured entry: a "SAVED!" moment plus quick enrichment actions (caption, emotion check, bookmark) so the user can add depth to the entry immediately, while it's still top of mind, without navigating away first.

## Entry points & navigation
- Entry: only reachable via `CaptureScreen`'s `onSaved` callback, which does `navigation.reset({ index: 1, routes: [{ name: 'MainTabs' }, { name: 'CaptureSaved', params: { entryId, previewFirstPhotoUri? } }] })` — so this screen is always the top of a 2-deep stack with `MainTabs` beneath it, never reachable via back-navigation into the camera.
- `route.params`: `entryId` (required), `previewFirstPhotoUri` (optional — used to render the card instantly before `AppContext`'s `entries` array has caught up with the just-created entry).
- Exit:
  - "HOME" button → `navigation.navigate('MainTabs', { screen: 'Home' })`.
  - "Write A Caption" → `navigation.navigate('NotesEditor', { entryId, editEntryDescription: true })`.
  - "Emotion check" → `navigation.navigate('EmotionCapture', { returnFrom: 'CaptureSaved', initialLabels: emotionLabels })` — see the handoff mechanism below for the return trip.

## Layout anatomy
1. "HOME" row, top-left — back-arrow icon + "HOME" label (not a true back button; always routes to Home explicitly).
2. Centered "SAVED!" title (Inter, black weight).
3. `FeedStyleEntryCard` (`src/components/common/FeedStyleEntryCard.tsx`) — the same full-bleed card component used in the Home feed, at a fixed 3:4 aspect ratio, showing the entry with its captured photo.
4. "Add more details:" label.
5. Action row: "Write A Caption" pill (pen icon) + "Emotion check" pill (heart icon), side by side.
6. "Bookmark" pill, centered below, full width of its own row — fills with a `LinearGradient` (`babyBlue → royalGold → fern`) when active instead of a plain border.

## States
- **`!displayEntry`** (entry not yet in `AppContext.entries` AND no `previewFirstPhotoUri` fallback was passed) → centered `LoadingSpinner`, nothing else rendered.
- **`displayEntry` via `previewFirstPhotoUri` fallback** (entry exists in params but hasn't propagated into `AppContext.entries` yet): a synthetic minimal `Entry`-shaped object is constructed inline (`type: 'experience'`, `isDraft: false`, `isBookmarked: false`, single photo) purely so the card can render immediately — this is a read-only stand-in, not a real entry; all the action buttons that require a real entry (`!entry && styles.actionButtonDisabled`) are disabled until the real `entry` resolves from context.
- **Bookmark toggled on/off**: local `isBookmarked` state flips immediately (optimistic) and calls `updateEntry({ ...entry, isBookmarked: next })` in the background; errors are only logged (`logger.error`), not surfaced to the user.
- **Returning from EmotionCapture**: see the handoff excerpt below — `pendingEmotionLabels`/`pendingEmotionNote` land in local state and trigger an `updateEntry` call, then are cleared from context so they don't reapply on a later focus.

## Data & context dependencies
- `useAppContext()` → `entries`, `updateEntry`, `pendingEmotionLabels`, `setPendingEmotionLabels`, `pendingEmotionNote`, `setPendingEmotionNote` — the last four implement the cross-screen "handoff" pattern documented in the root `CLAUDE.md` for `EmotionCaptureScreen` ↔ `CaptureScreen`/`EntryEditScreen`; this screen is a third participant in that same handoff.
- `useIsFocused()` — gates the emotion-handoff effect so it only fires while this screen is actually the visible one (prevents applying a stale pending-emotion write after the user has already navigated elsewhere and back).
- `useTheme()` → `colors`, passed through `createStyles(colors)`.

## Interactions
- No custom animations on this screen — everything is static `TouchableOpacity`s with `activeOpacity={0.7}`.
- Every action button is disabled (opacity 0.45) rather than hidden when `!entry`, so the layout doesn't shift once the real entry resolves.

## Theming notes
Standard `useTheme()` usage throughout; the only static brand-color usage is the bookmark-active gradient (`COLORS.babyBlue, COLORS.royalGold, COLORS.fern`) and the active-bookmark border tint (`COLORS.babyBlue`) — consistent with the "gradient = special/featured state" convention documented in `patterns/navigation-and-tab-bar.md` and the `screens/README.md` note about the capture action sheet.

## Reference UI skeleton
Small enough to skeleton almost completely — only the handoff effects and handlers are stripped to comments:

```tsx
// source: src/screens/CaptureSavedScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const CaptureSavedScreen = () => {
  const { colors } = useTheme();
  const { entryId, previewFirstPhotoUri } = route.params;
  const styles = useMemo(() => createStyles(colors), [colors]);

  // displayEntry: real `entry` from context, or a synthetic read-only stand-in built from
  // `previewFirstPhotoUri` while the real one hasn't propagated yet — see :53-70
  // emotion-handoff effect (pendingEmotionLabels/Note → updateEntry) — see :94-109
  // handleBookmarkToggle / handleCaptionPress / handleHome — see :113-133

  if (!displayEntry) return <View style={[styles.container, styles.centered]}><LoadingSpinner /></View>;

  const liveEntry = { ...displayEntry, isBookmarked };

  return (
    <View style={[styles.container, { paddingTop: safeTop }]}>
      <View style={styles.homeRow}>
        <TouchableOpacity style={styles.homeButton} onPress={handleHome}>
          <BackArrow color={colors.textPrimary} />
          <Text style={styles.homeButtonText}>HOME</Text>
        </TouchableOpacity>
      </View>

      <View style={styles.savedContentBlock}>
        <View style={styles.savedTitleWrap}>
          <Text style={styles.savedTitle}>SAVED!</Text>
        </View>

        <View style={[styles.cardWrapper, { paddingHorizontal: CARD_INSET }]}>
          <FeedStyleEntryCard entry={liveEntry} width={cardWidth} aspectRatio={3 / 4} directImageUri={previewFirstPhotoUri} />
        </View>

        <Text style={styles.addMoreLabel}>Add more details:</Text>

        <View style={styles.actionRow}>
          <TouchableOpacity style={[styles.actionButton, !entry && styles.actionButtonDisabled]} onPress={handleCaptionPress} disabled={!entry}>
            <PenIcon color={colors.textPrimary} />
            <Text style={styles.actionButtonText}>Write A Caption</Text>
          </TouchableOpacity>
          <TouchableOpacity style={[styles.actionButton, !entry && styles.actionButtonDisabled]} onPress={/* navigate EmotionCapture */ handleEmotionPress} disabled={!entry}>
            <HeartIcon color={colors.textPrimary} />
            <Text style={styles.actionButtonText}>Emotion check</Text>
          </TouchableOpacity>
        </View>

        <TouchableOpacity
          style={[styles.bookmarkButton, isBookmarked && styles.bookmarkButtonActive, !entry && styles.actionButtonDisabled]}
          onPress={handleBookmarkToggle}
          disabled={!entry}
        >
          {isBookmarked && (
            <LinearGradient
              colors={[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]}
              start={{ x: 0, y: 0 }} end={{ x: 1, y: 0 }}
              style={[StyleSheet.absoluteFillObject, { borderRadius: 24 * scaleFactor }]}
            />
          )}
          <CompassIcon color={colors.textPrimary} />
          <Text style={styles.actionButtonText}>Bookmark</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

function createStyles(colors) {
  return StyleSheet.create({
    container: { flex: 1, backgroundColor: colors.background },
    centered: { justifyContent: 'center', alignItems: 'center' },
    savedTitle: { fontFamily: FONTS.inter, fontWeight: FONT_WEIGHTS.black, color: colors.textPrimary, textAlign: 'center' },
    actionRow: { flexDirection: 'row', justifyContent: 'center', gap: 12, marginBottom: 10 },
    actionButton: { flexDirection: 'row', alignItems: 'center', justifyContent: 'center', gap: 6, borderWidth: 1, borderColor: colors.textPrimary, borderRadius: 24, paddingHorizontal: 14, paddingVertical: 9 },
    actionButtonDisabled: { opacity: 0.45 },
    bookmarkButton: { flexDirection: 'row', alignItems: 'center', justifyContent: 'center', gap: 6, borderWidth: 1, borderColor: colors.textPrimary, borderRadius: 24, paddingHorizontal: 20, paddingVertical: 9, alignSelf: 'center', overflow: 'hidden' },
    bookmarkButtonActive: { borderColor: COLORS.babyBlue }, // border tints to match the gradient fill it now contains
  });
}
```

**Key technique(s):** the emotion-capture return handoff (`:94-109`) reads `pendingEmotionLabels`/`pendingEmotionNote` from `AppContext` once (gated by `useIsFocused()`), applies them to local state + `updateEntry`, then clears them from context — so a later focus doesn't reapply a stale write. Every action button uses `disabled` + `actionButtonDisabled` opacity rather than conditional rendering, so the layout doesn't shift once the real `entry` resolves from context.

## Screenshot
_Not yet captured. TODO: capture light + dark: freshly-saved photo entry (default state), bookmark active (gradient fill), disabled-action state (entry not yet resolved from context — likely a very brief flash, may need a slowed-down dev build to catch)._

## Open questions / follow-ups
- `emotionNote` local state is set from `entry.emotionNote` and from the emotion-handoff effect, but nothing on this screen ever reads or displays `emotionNote` in the JSX — it's tracked but unused visually here. Worth confirming whether that's dead state or intended for a future UI element (e.g. showing the emotion note as a caption).
