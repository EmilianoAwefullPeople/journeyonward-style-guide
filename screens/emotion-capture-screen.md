# Emotion Capture

**Route name:** `EmotionCapture`
**File:** `src/screens/EmotionCaptureScreen.tsx` (thin wrapper) + `src/components/capture/EmotionalCaptureFlow.tsx` (the actual 2-step flow, ~250+ lines)
**Type:** Modal (`presentation: 'modal'`)

## Purpose
Captures how the user felt about an entry as a simple 3×3 emoji-grid multi-select plus an optional free-text note — a simplified, shipped version of the PRD's Russell's-Circumplex-Model emotion capture (valence/arousal sliders + 25-emotion grid). Serves the Reflective Traveler persona's emotional-tracking need.

## Entry points & navigation
- Entry: `CaptureSavedScreen`'s "Emotion check" (`{ returnFrom: 'CaptureSaved', initialLabels }`), `EntryEditScreen`'s "Emotion check" (`{ returnFrom: 'EntryEdit', initialLabels, entryId }`), and `EntryEditScreen`'s auto-deep-link when `focusField === 'emotion'`.
- `route.params`: `{ returnFrom?: 'Capture' | 'EntryEdit', initialLabels?: string[], entryId?: string }`. Per the file's own header comment, this screen deliberately uses `useNavigation()` (nearest navigator) rather than a root navigator ref, specifically so `goBack()` returns to whichever screen actually pushed it — from `Capture`'s nested stack, `goBack()` pops to `CaptureMain`; from the root stack, it pops to `EntryEdit`. `returnFrom` itself is currently unused for routing (noted in-file as "available for future pop-to-specific-screen if needed") — the actual return path is just `navigation.goBack()`.
- Exit:
  - Close (back arrow, top-left) → `navigation.goBack()`, discarding any selection.
  - Save (step 2's "Skip & Save", or step 1's advance) → `setPendingEmotionLabels(labels)` + `setPendingEmotionNote(note)` on `AppContext`, then `navigation.goBack()`. This is the write side of the handoff mechanism documented in the root `CLAUDE.md` and consumed by `CaptureSavedScreen`/`EntryEditScreen`'s `pendingEmotionLabels` effects (see those docs).

## Layout anatomy
`EmotionCaptureScreen` itself is just a back button + safe-area padding wrapper around `EmotionalCaptureFlow`, which renders:
- **Step 1**: title "How would you describe your mood?" + a 3×3 grid of emoji+label cells (`EMOTION_CAPTURE_OPTIONS`, `src/utils/emotionUtils.ts`), multi-select, staggered fade-in (30ms per cell) on mount; a white pill "Next" button below, disabled state implied by `nextButtonDisabled` style (though the actual disable condition isn't gated on selection count in the excerpt reviewed — advancing with zero selections may be allowed).
- **Step 2**: back arrow (returns to step 1) + a summary line ("You selected: X, Y" / "No emotions selected") + a multiline note `TextInput` (`EMOTION_NOTE_MAX_LENGTH = 200`) + a white pill "Skip & Save" button.

## States
- **`step` (1 | 2)**: cross-fades between steps via a single shared `stepOpacity` Animated value (fade to 0, swap step, reset to 0, fade to 1 — 220ms each leg, `STEP_TRANSITION_MS`), rather than two views ever being mounted simultaneously.
- **Per-cell selected/unselected**: each of the 9 grid cells has its own `selectionProgress` Animated value (0/1) driving an interpolated label color (`colors.textSecondary → colors.textPrimary`); `LayoutAnimation.configureNext(easeInEaseOut)` also fires on every toggle for any layout shift.
- **Selection persists across steps**: `selectedLabels` is shared state across both steps — going back to step 1 from step 2 doesn't lose selections.
- **Note capped**: input enforces 200 chars at save time (`.slice(0, EMOTION_NOTE_MAX_LENGTH)`), trimmed and deduped (`Array.from(new Set(...))`) before calling `onSave`.
- **`initialLabels`**: pre-seeds `selectedLabels` (and their `selectionProgress` values start already at 1) when returning to re-edit an entry's existing emotions.

## Data & context dependencies
- `useAppContext()` → `setPendingEmotionLabels`, `setPendingEmotionNote` only — this screen never reads or writes an `Entry` directly; it's purely a labels-collector that hands its result back through context for the calling screen to actually persist.
- `useTheme()` → `colors`.
- `EMOTION_CAPTURE_OPTIONS` (`src/utils/emotionUtils.ts`) — the 9 emoji/label pairs; not the PRD's full 25-emotion valence×arousal grid.

## Interactions
- Grid cell tap: `handleToggle` — toggles membership in `selectedLabels`, triggers a `LayoutAnimation` pass and the per-cell color interpolation.
- Step transition: opacity-only cross-fade, no slide/translate.
- Grid entrance: per-cell staggered fade (`gridStagger[i]`, `delay: i * 30`) only on step 1 becoming active.

## Theming notes
`useTheme()` for text/background colors; the two CTA buttons ("Next", "Skip & Save") are hardcoded `COLORS.white` background with `COLORS.black` text regardless of theme — a deliberate "always-light pill button" treatment rather than following dark mode, consistent with high-contrast CTA styling seen elsewhere (e.g. onboarding's white CTA pill).

## Reference UI skeleton
`EmotionCaptureScreen` is a thin wrapper (shown in full); `EmotionalCaptureFlow`'s two steps are the real content — step 1's 3×3 grid is shown as one representative row (rows 2 and 3 repeat the identical cell structure for indices 3-5 and 6-8):

```tsx
// source: src/screens/EmotionCaptureScreen.tsx + src/components/capture/EmotionalCaptureFlow.tsx (skeleton — business logic stripped, see cited lines for real implementation)

// --- EmotionCaptureScreen (thin wrapper, full) ---
const EmotionCaptureScreen = () => {
  const { initialLabels = [] } = route.params ?? {};
  const handleClose = () => navigation.goBack();
  const handleSave = (labels, note) => { /* setPendingEmotionLabels/Note on AppContext, then goBack() — :36-41 */ };

  return (
    <View style={[styles.container, { paddingTop: insets.top }]}>
      <TouchableOpacity style={styles.backButton} onPress={handleClose}>
        <BackArrow color={colors.textPrimary} />
      </TouchableOpacity>
      <View style={[styles.flowWrapper, { flex: 1, paddingTop: 45 + spacingBelowClose }]}>
        <EmotionalCaptureFlow initialLabels={initialLabels} onClose={handleClose} onSave={handleSave} />
      </View>
    </View>
  );
};

// --- EmotionalCaptureFlow ---
const EmotionalCaptureFlow = ({ initialLabels, onClose, onSave }) => {
  const [step, setStep] = useState(1); // crossfades via shared stepOpacity, see Key technique(s)
  const [selectedLabels, setSelectedLabels] = useState(initialLabels);
  const [emotionNote, setEmotionNote] = useState('');
  const selectionProgress = useRef(EMOTION_CAPTURE_OPTIONS.map((_, i) => new Animated.Value(initialLabels.includes(EMOTION_CAPTURE_OPTIONS[i].label) ? 1 : 0))).current;
  const gridStagger = useRef(EMOTION_CAPTURE_OPTIONS.map(() => new Animated.Value(0))).current; // staggered fade-in, delay: i * 30ms

  if (step === 1) {
    return (
      <Animated.View style={[styles.container, { opacity: stepOpacity }]}>
        <ScrollView style={styles.scroll} contentContainerStyle={styles.scrollContent}>
          <Text style={styles.title}>How would you{'\n'}describe your mood?</Text>

          {/* 3 rows × 3 cols = 9 cells total; row shown once, repeat for indices [3,4,5] and [6,7,8] */}
          <View style={styles.gridRow}>
            {[0, 1, 2].map((col) => {
              const item = EMOTION_CAPTURE_OPTIONS[col];
              const isSelected = selectedLabels.includes(item.label);
              return (
                <Animated.View key={item.label} style={[styles.gridCell, { opacity: gridStagger[col] }]}>
                  <TouchableOpacity style={styles.gridCellTouchable} onPress={() => handleToggle(item.label)}>
                    <Text style={styles.gridEmoji}>{item.emoji}</Text>
                    <Animated.Text
                      style={[styles.gridLabel, isSelected && styles.gridLabelSelected, { color: selectionProgress[col].interpolate({ inputRange: [0, 1], outputRange: [colors.textSecondary, colors.textPrimary] }) }]}
                    >
                      {item.label}
                    </Animated.Text>
                  </TouchableOpacity>
                </Animated.View>
              );
            })}
          </View>
          {/* ...gridRow for [3,4,5], gridRow for [6,7,8] — identical structure */}

          <TouchableOpacity style={[styles.nextButton, selectedLabels.length === 0 && styles.nextButtonDisabled]} onPress={goToStep2} disabled={selectedLabels.length === 0}>
            <Text style={styles.nextButtonText}>Next</Text>
          </TouchableOpacity>
        </ScrollView>
      </Animated.View>
    );
  }

  const selectedSummary = selectedLabels.length > 0 ? `You selected: ${selectedLabels.join(', ')}` : 'No emotions selected';

  return (
    <Animated.View style={[styles.container, { flex: 1, opacity: stepOpacity }]}>
      <View style={styles.step2Header}>
        <TouchableOpacity style={styles.backButton} onPress={goToStep1}>
          <BackArrow color={colors.textPrimary} />
        </TouchableOpacity>
      </View>
      <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
        <ScrollView style={styles.scroll} contentContainerStyle={styles.scrollContentStep2}>
          <Text style={styles.step2Summary}>{selectedSummary}</Text>
          <TextInput
            style={styles.noteInput}
            placeholder="Describe the feeling in a few words"
            value={emotionNote}
            onChangeText={(t) => setEmotionNote(t.slice(0, EMOTION_NOTE_MAX_LENGTH))}
            multiline
            maxLength={EMOTION_NOTE_MAX_LENGTH}
          />
          <TouchableOpacity style={styles.skipSaveButton} onPress={handleSave}>
            <Text style={styles.skipSaveButtonText}>{emotionNote.trim() ? 'Save' : 'Skip & Save'}</Text>
          </TouchableOpacity>
        </ScrollView>
      </KeyboardAvoidingView>
    </Animated.View>
  );
};
```

**Key technique(s):** the shared-opacity step-crossfade (`goToStep2`/`goToStep1`, `:199-213`) — fade `stepOpacity` to 0, swap `step` state, reset to 0, fade back to 1 (220ms each leg) — is a lighter-weight alternative to mounting/unmounting two separate screens when a flow only has two steps; only one step's tree is ever mounted. The two CTA buttons ("Next", "Skip & Save") are hardcoded `COLORS.white`/`COLORS.black` regardless of theme, not `useTheme()` tokens.

## Screenshot
_Not yet captured. TODO: capture light + dark: step 1 with 0 and 3+ selections, step 2 with a note typed, step 2 with "No emotions selected" summary._

## Open questions / follow-ups
- The PRD (Feature 6 / Appendix B) describes valence/arousal sliders mapping into a 25-emotion grid with custom "Other..." emotion entry; the shipped implementation is a fixed 3×3 (9-option) emoji grid with a free-text note instead — no sliders, no 25-emotion mapping, no explicit "Other" button (though the free-text note likely serves that purpose). Worth confirming with product whether the PRD is aspirational/outdated here or whether this is a deliberate simplification for the current build.
- `returnFrom` is accepted as a param and documented in a comment as "available for future...if needed" but has no effect on current behavior — dead/forward-looking param, not a bug, but worth knowing it doesn't do what its name implies yet.
