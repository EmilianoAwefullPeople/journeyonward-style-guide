# Entry Edit

**Route name:** `EntryEdit`
**File:** `src/screens/EntryEditScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`)

## Purpose
The "add more details to an existing entry" screen — structurally almost identical to `CaptureSavedScreen` (same card + caption/emotion/bookmark actions), but for revisiting an already-saved entry later rather than right after capture. Every action persists immediately via `updateEntry`; there is no separate "Save" step.

## Entry points & navigation
- Entry: `EntryDetailScreen`'s 3-dot menu → "Edit" (`navigate('EntryEdit', { entryId })`).
- `route.params`: `{ entryId: string, focusField?: 'title' | 'description' | 'emotion' }` — `focusField` lets a caller deep-link straight into a sub-editor (see below) instead of landing on the plain edit screen.
- Exit / further navigation:
  - Back arrow → `navigation.goBack()`.
  - "Write A Caption" → `navigate('NotesEditor', { entryId, editEntryDescription: true })`.
  - "Emotion check" → `navigate('EmotionCapture', { returnFrom: 'EntryEdit', initialLabels: emotionLabels, entryId })`.
  - **Auto-navigate on first focus** if `focusField` is set: `title`/`description` → jumps straight into NotesEditor; `emotion` → jumps straight into EmotionCapture — then clears the param (`navigation.setParams({ focusField: undefined })`) so re-focusing this screen later doesn't re-trigger the jump. Fires after a 100ms `setTimeout`, guarded by `hasHandledInitialFocusRef` so it only ever fires once per `entryId`.

## Layout anatomy
Identical structure to `CaptureSavedScreen` minus the "SAVED!" title:
1. Back button row, top-left.
2. `FeedStyleEntryCard` at 3:4 aspect ratio.
3. "Add more details:" label.
4. Action row: "Write A Caption" (pen icon) + "Emotion check" (heart icon).
5. "Bookmark" pill, gradient-filled when active.

## States
- **`!entry`** (entry not found / not yet loaded) → centered `LoadingSpinner`, everything else unrendered — unlike `CaptureSavedScreen`, there's no `previewFirstPhotoUri` fallback here, since this screen is only reached for entries that already exist.
- **Bookmark toggle**: optimistic local flip + background `updateEntry` (errors logged only, not surfaced).
- **`focusField` deep-link pending**: transient — the screen renders normally for one tick, then the effect fires and navigates away; a user would only see the plain edit screen flash briefly if `focusField` is set (worth checking in a live screenshot whether this flash is noticeable).
- **Emotion-handoff return**: identical pattern to `CaptureSavedScreen` — `pendingEmotionLabels`/`pendingEmotionNote` from context are consumed once, applied to local state, persisted via `updateEntry`, then cleared from context.

## Data & context dependencies
- `useAppContext()` → `entries`, `updateEntry`, `pendingEmotionLabels`, `setPendingEmotionLabels`, `pendingEmotionNote`, `setPendingEmotionNote`.
- `useIsFocused()` gates the emotion-handoff effect exactly as in `CaptureSavedScreen`.
- `useTheme()` → `colors`, via the same `createStyles(colors)` factory pattern.

## Interactions
- No custom animation — static `TouchableOpacity`s, `activeOpacity={0.7}`.

## Theming notes
Same conventions as `CaptureSavedScreen`: theme tokens throughout, static brand-gradient (`COLORS.babyBlue/royalGold/fern`) only for the active-bookmark fill and border tint.

## Reference UI skeleton
Structurally near-identical to `CaptureSavedScreen` minus the "SAVED!" header — see that doc's skeleton for the shared card/action-row/bookmark JSX. Only what differs is expanded here:

```tsx
// source: src/screens/EntryEditScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const EntryEditScreen = () => {
  const { entryId, focusField } = route.params; // focusField?: 'title' | 'description' | 'emotion'
  const entry = useMemo(() => entries.find(e => e.id === entryId), [entries, entryId]);
  const styles = useMemo(() => createStyles(colors), [colors]);

  // reset hasHandledInitialFocusRef when entryId changes — :75-77
  // emotion-handoff effect, same shape as CaptureSavedScreen — :80-93
  // one-shot deep-link-to-subeditor effect (focusField → NotesEditor/EmotionCapture, 100ms) — :96-114

  if (!entry) return <View style={[styles.container, styles.centered]}><LoadingSpinner /></View>; // no previewFirstPhotoUri fallback — entry must already exist

  const liveEntry = { ...entry, isBookmarked };

  return (
    <View style={[styles.container, { paddingTop: safeTop }]}>
      <View style={styles.backRow}>
        <TouchableOpacity style={styles.backButton} onPress={() => navigation.goBack()}>
          <BackArrow color={colors.textPrimary} />
        </TouchableOpacity>
      </View>

      <View style={styles.contentBlock}>
        <View style={[styles.cardWrapper, { paddingHorizontal: CARD_INSET }]}>
          <FeedStyleEntryCard entry={liveEntry} width={cardWidth} aspectRatio={3 / 4} />
        </View>

        <Text style={styles.addMoreLabel}>Add more details:</Text>

        <View style={styles.actionRow}>
          <TouchableOpacity style={styles.actionButton} onPress={handleCaptionPress}>
            <PenIcon color={colors.textPrimary} />
            <Text style={styles.actionButtonText}>Write A Caption</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.actionButton} onPress={() => navigation.navigate('EmotionCapture', { returnFrom: 'EntryEdit', initialLabels: emotionLabels, entryId })}>
            <HeartIcon color={colors.textPrimary} />
            <Text style={styles.actionButtonText}>Emotion check</Text>
          </TouchableOpacity>
        </View>

        <TouchableOpacity style={[styles.bookmarkButton, isBookmarked && styles.bookmarkButtonActive]} onPress={handleBookmarkToggle}>
          {isBookmarked && (
            <LinearGradient colors={[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]} start={{ x: 0, y: 0 }} end={{ x: 1, y: 0 }} style={[StyleSheet.absoluteFillObject, { borderRadius: 24 * scaleFactor }]} />
          )}
          <CompassIcon color={colors.textPrimary} />
          <Text style={styles.actionButtonText}>Bookmark</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

// createStyles(colors) — identical shape to CaptureSavedScreen's (see that doc); notably no
// actionButtonDisabled usage here since every button is always enabled once `entry` exists.
```

**Key technique(s):** the one-shot deep-link-to-subeditor effect (`:96-114`) is the only behavior that differs meaningfully from `CaptureSavedScreen` — on first focus, if `focusField` was passed, it auto-navigates straight into the matching sub-editor after a 100ms delay, then clears the param via `navigation.setParams({ focusField: undefined })` so re-focusing later doesn't re-trigger it, guarded by `hasHandledInitialFocusRef` for one-shot-per-`entryId` semantics.

## Screenshot
_Not yet captured. TODO: capture light + dark: default edit view, bookmark active, loading (entry-not-yet-resolved) state. A `focusField`-triggered deep link is transient and may not be worth a static screenshot — a screen recording would show the brief flash (if any) better._

## Open questions / follow-ups
- Because this screen is nearly a duplicate of `CaptureSavedScreen` (same card, same three actions, same handoff pattern) minus the "SAVED!" header and the `previewFirstPhotoUri` fallback, it may be a candidate for extracting a shared component/hook if the two ever need to evolve together — flagging for an engineer decision, not something to guess at from the design-reference pass alone.
