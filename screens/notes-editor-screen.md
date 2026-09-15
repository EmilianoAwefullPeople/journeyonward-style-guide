# Notes Editor

**Route name:** `NotesEditor`
**File:** `src/screens/NotesEditorScreen.tsx`
**Type:** Transparent modal (`presentation: 'transparentModal'`, `animation: 'none'`, `cardStyle: { backgroundColor: 'transparent' }`) — the screen drives its own enter/exit animation instead of relying on the navigator's transition.

## Purpose
A single dual-purpose full-screen text editor with two distinct modes selected by params:
1. **Journal mode** (default): write a brand-new journal/insight entry, optionally linked to an experience (branch journal) or a specific album.
2. **Caption mode** (`editEntryDescription: true` + `entryId`): edit an existing entry's title + description in place.

Serves the Reflective Traveler (journal mode) and both the Live-Capture/Retrospective personas (caption mode, via `CaptureSaved`/`EntryEdit`'s "Write A Caption" action).

## Entry points & navigation
- Entry points (journal mode): `HomeScreen`'s floating pen icon (`{ journalPlaceholder }`), `EntryDetailScreen`'s "Write About This Experience" button and its 3-dot "Journal" option (`{ entryId }` — creates a branch journal), trip-album flows passing `{ albumId }`.
- Entry points (caption mode): `CaptureSavedScreen`/`EntryEditScreen`'s "Write A Caption" action (`{ entryId, editEntryDescription: true }`).
- `route.params`: `{ entryId?, albumId?, journalPlaceholder?, editEntryDescription? }`.
- Exit:
  - Close (×) button → if there are unsaved changes, confirms via `Alert` ("Discard changes?") before `performClose()`; otherwise closes immediately. `performClose` animates out (400ms) then calls `navigation.goBack()` after a 200ms delay (not synced to the animation's actual completion — see Open questions).
  - **Save, journal mode**: builds a new `Journal` (UUID, content, timestamp, best-effort location + reverse-geocoded city/country, `albumId` from params or the active trip, `linkedEntryId` if `entryId` was passed), calls `createJournal`, then (if linked to an entry) updates that entry's `journalId`, then `navigation.replace('JournalDetail', { journalId })` — replaces this screen in the stack rather than pushing on top of it.
  - **Save, caption mode**: `updateEntry({ ...entry, title, description })`, then `navigation.goBack()` (or `navigate('CaptureSaved', { entryId })` as a fallback if there's nothing to go back to) — deliberately immediate rather than routed through the animated `performClose`, per an inline comment about reliability with `transparentModal` + async `refreshData()`.

## Layout anatomy
1. Full-bleed background `View` (`colors.background`) inset by safe area, wrapped in a `TouchableWithoutFeedback` that dismisses the keyboard on tap-outside.
2. Close (×) button, top-right, `zIndex: 1003`.
3. Input area:
   - **Caption mode only**: a single-line title `AppTextInput` ("Add a title", max 100 chars, `returnKeyType="next"` chains focus into the body field).
   - Body `AppTextInput`, multiline, placeholder text depends on mode (`'Write a caption...'` / `'Write About This Experience...'` if `entryId` present / the rotating prompt passed from Home / the generic default), max length 500 (caption) or 2000 (journal).
4. "Save" button — floats above the keyboard when open (`keyboardHeight + 16` from bottom) or above the safe area when closed (`insets.bottom + 80`), fades in/out (`saveButtonOpacity`) based on whether there's anything worth saving.

## States
- **Journal vs. caption mode** (`isCaptionMode = editEntryDescription && !!entryId`): swaps placeholder text, whether the title field renders at all, max character counts, and the entire save/exit behavior described above.
- **Caption mode, initial seed**: on first data arrival, seeds `titleText`/`noteText` from the entry's current `title`/`description` exactly once (`captionSeededRef`), so a later `refreshData()` re-render doesn't clobber in-progress edits.
- **Save button visibility** (`saveButtonOpacity`, 200ms fade): journal mode shows it once body text is non-empty; caption mode shows it if either field is non-empty OR differs from its seeded baseline (so clearing a previously-set title/caption back to non-empty-but-unchanged still counts as "has content to save").
- **Save button disabled** (separate from visibility): caption mode disables if neither field has content nor differs from baseline; journal mode disables if the body is empty.
- **Keyboard open/closed** (`isKeyboardOpen`, iOS `keyboardWillShow`/`keyboardWillHide` listeners): repositions the floating Save button to sit just above the keyboard.
- **Unsaved-changes check on close**: caption mode compares both fields against their baselines; journal mode just checks whether the body has any trimmed text — either triggers the "Discard changes?" confirmation.

## Data & context dependencies
- `useAppContext()` → `createJournal`, `updateEntry`, `entries`, `activeTripAlbum`.
- Dynamic `import('expo-location')` — best-effort last-known position + reverse geocoding at journal save time only (not caption mode); silently proceeds without location if permission isn't granted or the call fails.
- `generateUUID()` (`src/utils/uuid.ts`) for new journal IDs.
- Custom `AppText`/`AppTextInput` components (`src/components/common/`) rather than raw RN `Text`/`TextInput` — per `CLAUDE.md`'s component list, likely wrapping font-scaling/accessibility defaults app-wide.

## Interactions
- Entrance: `screenOpacity` (0→1) + `contentTranslateY` (20→0), both 550ms with the same bezier — the comment notes this is deliberately "synced with HomeScreen's expand animation" (the notes-transition pill-to-sheet animation described in `home-screen.md`), since this modal is the destination of that transition.
- Exit (`performClose`): reverse animation (opacity→0, translateY→50, 400ms) fired in parallel with a `setTimeout(() => navigation.goBack(), 200)` — the actual navigation happens on a fixed timer, not an animation-finished callback, so it's not guaranteed to be visually in sync with the animation on slower devices.
- Auto-focus: body input focuses automatically 350ms after mount (`setTimeout`), timed to occur mid-entrance-animation rather than instantly.
- Tap-outside-input (on the background `TouchableWithoutFeedback`) blurs the active input without closing the screen; the input wrapper and close button both have their own nested `TouchableWithoutFeedback`/`TouchableOpacity` to stop that tap from propagating to the dismiss-keyboard handler.

## Theming notes
Standard `useTheme()` usage; no static brand colors on this screen at all — even the Save button uses `colors.surface`/`colors.textPrimary` rather than a gradient, unlike the bookmark/featured-card convention used elsewhere.

## Reference UI skeleton
Full render tree — real structure end to end, business logic (mode detection, save/close handlers, keyboard listeners) reduced to comments:

```tsx
// source: src/screens/NotesEditorScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const NotesEditorScreen = () => {
  const { entryId, albumId, journalPlaceholder, editEntryDescription } = route.params;
  const isCaptionMode = editEntryDescription && !!entryId;

  // titleText/noteText state + baselines (captionSeededRef seeds from entry once) — see file top
  // handleClose (Alert confirm if unsaved) / handleSave (createJournal or updateEntry, see Purpose) — see :19-21 (doc's own Entry-points section)
  // isKeyboardOpen/keyboardHeight tracking, auto-focus 350ms after mount — see :121 area

  const styles = useMemo(() => StyleSheet.create({
    container: { flex: 1, backgroundColor: 'transparent' },
    notesContainer: { flex: 1, backgroundColor: colors.background, paddingTop: insets.top + 60, paddingBottom: insets.bottom + 20, paddingHorizontal: 16 },
    closeButtonWrapper: { position: 'absolute', top: insets.top + 20, right: 16, zIndex: 1003 },
    closeText: { fontSize: 45, color: colors.textPrimary, fontFamily: FONTS.inter, lineHeight: 45 },
    inputWrapper: { flex: 1, marginTop: 20, marginBottom: 110 },
    titleInput: { color: colors.textPrimary, fontSize: 20, fontWeight: FONT_WEIGHTS.bold, fontFamily: FONTS.inter, lineHeight: 28, paddingLeft: 16 },
    textInput: { flex: 1, color: colors.textPrimary, fontSize: 17, fontFamily: FONTS.inter, lineHeight: 24, paddingLeft: 16 },
    saveButton: { backgroundColor: colors.surface, borderRadius: 20, paddingHorizontal: 16, paddingVertical: 8, minWidth: 60, alignItems: 'center' },
    saveButtonText: { color: colors.textPrimary, fontSize: 16, fontWeight: FONT_WEIGHTS.semibold, fontFamily: FONTS.inter },
  }), [colors, insets.top, insets.bottom]);

  const saveButtonDisabled = isCaptionMode
    ? !(titleText.trim() || noteText.trim() || titleText !== (titleBaselineRef.current ?? '') || noteText !== (captionBaselineRef.current ?? ''))
    : noteText.trim().length === 0;

  return (
    <Animated.View style={[styles.container, { opacity: screenOpacity, transform: [{ translateY: contentTranslateY }] }]}>
      <TouchableWithoutFeedback onPress={handleBlurOverlayPress}>
        <View style={styles.notesContainer}>
          <View style={styles.closeButtonWrapper}>
            <TouchableOpacity style={styles.closeButton} onPress={handleClose}>
              <AppText style={styles.closeText}>×</AppText>
            </TouchableOpacity>
          </View>

          <TouchableWithoutFeedback>
            <View style={styles.inputWrapper}>
              {isCaptionMode && (
                <AppTextInput
                  style={styles.titleInput}
                  placeholder="Add a title"
                  value={titleText}
                  onChangeText={handleTitleChange}
                  returnKeyType="next"
                  maxLength={100}
                  onSubmitEditing={() => notesInputRef.current?.focus()}
                />
              )}
              <AppTextInput
                ref={notesInputRef}
                style={styles.textInput}
                placeholder={
                  isCaptionMode ? 'Write a caption...'
                  : entryId ? 'Write About This Experience...'
                  : journalPlaceholder || 'Type and document it here...'
                }
                value={noteText}
                onChangeText={handleTextChange}
                multiline
                maxLength={isCaptionMode ? 500 : 2000}
              />
            </View>
          </TouchableWithoutFeedback>

          <Animated.View style={{ opacity: saveButtonOpacity, position: 'absolute', right: 16, bottom: isKeyboardOpen ? keyboardHeight + 16 : insets.bottom + 80 }}>
            <TouchableOpacity style={styles.saveButton} onPress={handleSave} disabled={saveButtonDisabled}>
              <AppText style={styles.saveButtonText}>Save</AppText>
            </TouchableOpacity>
          </Animated.View>
        </View>
      </TouchableWithoutFeedback>
    </Animated.View>
  );
};
```

**Key technique(s):** the caption-mode "save visibility differs from disabled" split (`:278-291`) — `saveButtonOpacity` fades in whenever either field has content OR differs from its seeded baseline, but the button is only `disabled` under the stricter of those same conditions — so clearing a previously-set field back to non-empty-but-unchanged still shows Save without necessarily being tappable in every intermediate state. The two checks are intentionally not identical; read `:278-291` before touching either.

## Screenshot
_Not yet captured. TODO: capture light + dark: journal mode empty/typing, journal mode with rotating placeholder text, caption mode seeded from an existing entry, save button visible vs. hidden, keyboard-open layout._

## Open questions / follow-ups
- `performClose`'s `setTimeout(..., 200)` for `navigation.goBack()` is a fixed delay racing against a 400ms animation — on a slow device the screen could visually "jump" before the fade-out finishes, or on a fast device the nav could fire before the animation is visually complete. Worth a screen recording to check whether this is perceptible, and if so, whether it should instead use the animation's `.start(callback)`.
- Journal mode's location capture is silently best-effort with no user-facing indicator that it's happening or that it failed — consistent with the app's general "fail silently, fall back gracefully" pattern for location elsewhere (e.g. `HomeScreen`'s sunset greeting), but worth flagging to design as a place where an entry could end up geo-tagged or not with no visible reason to the user.
