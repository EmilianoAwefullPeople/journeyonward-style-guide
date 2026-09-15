# Unpack Trip Details Screen

**Route name:** `UnpackTripDetails`
**File:** `src/screens/unpack/UnpackTripDetailsScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
Stage 1 of 4 in the "Unpack A Trip" retrospective-import wizard (serves the Retrospective Traveler persona): collects a trip name, start date, end date, and optional description as a 4-step inner wizard on one screen. Deliberately does **not** persist a `TripAlbum` yet — only `UnpackHighlightScreen` (Stage 3a) creates the album record, once the user has committed to keeping at least some photos, so an abandoned flow never leaves an empty album behind.

## Entry points & navigation
- **Path A (default)**: from the global "+" action sheet in `AppNavigator.tsx` (`handlePlusMenuAction('UnpackTripDetails')`, "Unpack A Trip" featured card — hidden while travel mode is active, since unpacking is a post-trip action). No params.
- **Path B (photos-first)**: reached via a navigation *reset* from `UnpackProcessingScreen` when the user started from `UnpackAddMemories` directly, with `route.params.suggestions` (already-clustered photos) and inferred `inferredStartDate`/`inferredEndDate`. In this path the step bar is offset by one (see States).
- On save (step 4): Path A → `navigation.navigate('UnpackAddMemories', { albumId, tripStartDate, tripEndDate, albumName, albumDescription })`. Path B → `navigation.navigate('UnpackHighlight', { albumId, suggestions, albumName, albumStartDate, albumEndDate, albumDescription })`.
- Back on step 1 calls `navigation.goBack()` (exits the flow); back on steps 2-4 decrements the inner `step` state instead of navigating.
- An alternate exit from step 1: "Or start by importing photos" link → `navigation.navigate('UnpackAddMemories', {})`, skipping trip-detail collection entirely (this is how Path B gets initiated).

## Layout anatomy
1. `MeshGradientBackground` full-screen backdrop.
2. Header: back button (chevron on steps 2-4, "×" close on step 1) + `UnpackStepBar` (shared step-progress indicator component).
3. Scrollable body, one of four inner steps rendered at a time:
   - **Step 1** — "Give your trip a name": optional context line (Path B only, moment count), title `TextInput`, conditional "Next →" CTA (appears only once non-empty), and (Path A only) an "Or start by importing photos" alt-path link below a divider.
   - **Step 2** — "When did it start?": a `DateTimePicker` (spinner on iOS), "Next →" CTA.
   - **Step 3** — "When did it end?" (bounded `minimumDate = startDate`): same pattern.
   - **Step 4** — "Anything to say about it?": multiline description `TextInput`, CTA text changes between "Start unpacking →" and "Skip & continue →" depending on whether text was entered.

## States
- `step: 1 | 2 | 3 | 4` — drives which of the four render functions is shown; this is the screen's primary state machine.
- `isPhotosFirst` (derived from `Array.isArray(route.params?.suggestions)`) — toggles the Path A / Path B branches described above (alt-path link visibility, step-bar stage offset, and final-step navigation target).
- Keyboard-open state (`isKeyboardOpen`, `keyboardHeight`) — adjusts bottom content padding so the active input/CTA stays clear of the keyboard.
- `canProceedStep1` — the step-1 CTA only renders once `title.trim().length > 0`.
- End-date auto-correction: a `useEffect` clamps `endDate` up to `startDate` whenever `startDate` changes past it.

## Data & context dependencies
- No `AppContext`/`ThemeContext` data reads beyond `useTheme()` for `colors`/`isDark`.
- Generates the album's `id` locally via `generateUUID()` — the ID is created here but the row isn't written until Stage 3a.
- Shared `UnpackStepBar` component (`src/components/unpack/`) renders progress; `currentStage` passed as `isPhotosFirst ? 2 : 1`.

## Interactions
- Standard `TextInput`/`DateTimePicker` interactions; no gestures or animations beyond `KeyboardAvoidingView`'s built-in behavior.
- `TouchableWithoutFeedback` wraps steps 1 and 4 to dismiss the keyboard on background tap.

## Theming notes
- Fully theme-aware via `useTheme()` (`colors.background`, `colors.textPrimary`, `colors.textSecondary`, `colors.buttonGray`, `colors.border`).
- `MeshGradientBackground` takes `lightMode={!isDark}` explicitly rather than deriving it internally.

## Reference UI skeleton
Whole render tree, business logic stripped. Four `renderStepN()` functions, one shown at a time based on `step`.

```tsx
// source: src/screens/unpack/UnpackTripDetailsScreen.tsx (skeleton — business logic stripped)

const renderStep1 = () => (
  <TouchableWithoutFeedback onPress={() => Keyboard.dismiss()}>
    <View style={styles.stepContent}>
      <Text style={styles.question}>Give your trip a name</Text> {/* Playfair Display italic bold, 22px */}
      {isPhotosFirst && suggestionsCount > 0 && (
        <Text style={styles.contextLine}>{count} moment{s} found and ready to highlight.</Text>
      )}
      <TextInput ref={titleInputRef} style={styles.input} value={title} onChangeText={setTitle} maxLength={120} />
      {canProceedStep1 && (
        <TouchableOpacity style={styles.cta} onPress={handleNext}><Text style={styles.ctaText}>Next →</Text></TouchableOpacity>
      )}
      {!isPhotosFirst && (
        <View style={styles.altPathWrap}> {/* marginTop: SPACING.xl, center, gap: SPACING.md */}
          <View style={styles.altPathDivider} />
          <TouchableOpacity onPress={() => navigation.navigate('UnpackAddMemories', {})}>
            <Text style={styles.altPathText}>Or start by importing photos</Text> {/* underlined */}
          </TouchableOpacity>
        </View>
      )}
    </View>
  </TouchableWithoutFeedback>
);

const renderStep2 = () => ( // "When did it start?" — identical shape to step 3
  <View style={styles.stepContent}>
    <Text style={styles.question}>When did it start?</Text>
    <View style={styles.pickerContainer}>
      <DateTimePicker value={startDate} mode="date" display="spinner" maximumDate={new Date()} onChange={(_, d) => d && setStartDate(d)} />
    </View>
    <TouchableOpacity style={styles.cta} onPress={handleNext}><Text style={styles.ctaText}>Next →</Text></TouchableOpacity>
  </View>
);

const renderStep3 = () => ( // "When did it end?" — same shape, minimumDate={startDate}
  <View style={styles.stepContent}>
    <Text style={styles.question}>When did it end?</Text>
    <View style={styles.pickerContainer}>
      <DateTimePicker value={endDate} mode="date" display="spinner" minimumDate={startDate} maximumDate={new Date()} onChange={(_, d) => d && setEndDate(d)} />
    </View>
    <TouchableOpacity style={styles.cta} onPress={handleNext}><Text style={styles.ctaText}>Next →</Text></TouchableOpacity>
  </View>
);

const renderStep4 = () => (
  <TouchableWithoutFeedback onPress={() => Keyboard.dismiss()}>
    <View style={[styles.stepContent, { paddingBottom: contentBottomPadding }]}>
      <Text style={styles.question}>Anything to say about it?</Text>
      <TextInput style={styles.descriptionInput} value={description} onChangeText={setDescription} multiline maxLength={2000} />
      {/* handleSave: generates albumId locally, does NOT persist — see "Key technique" */}
      <TouchableOpacity style={styles.cta} onPress={handleSave}>
        <Text style={styles.ctaText}>{description.trim() ? 'Start unpacking →' : 'Skip & continue →'}</Text>
      </TouchableOpacity>
    </View>
  </TouchableWithoutFeedback>
);

// ── Screen ───────────────────────────────────────────────────────────────────
return (
  <KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'} style={styles.container}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
    <View style={[styles.header, { paddingTop: insets.top + SPACING.sm }]}>
      <View style={styles.headerRow}>
        <TouchableOpacity onPress={handleBack} style={styles.backButton}>
          {step === 1 ? <Ionicons name="close" size={22} /> : <Text style={styles.backText}>← Back</Text>}
        </TouchableOpacity>
      </View>
      <UnpackStepBar currentStage={isPhotosFirst ? 2 : 1} />
    </View>
    <ScrollView keyboardShouldPersistTaps="handled" contentContainerStyle={[styles.scrollContent, { paddingBottom: contentBottomPadding }]}>
      {step === 1 && renderStep1()}
      {step === 2 && renderStep2()}
      {step === 3 && renderStep3()}
      {step === 4 && renderStep4()}
    </ScrollView>
  </KeyboardAvoidingView>
);
```

**Key technique(s):** The two-path branch inside `handleSave` (step 4) is the crux of why this screen never writes to the database itself — the album `id` is generated locally via `generateUUID()`, but only `albumId` + form fields travel forward as nav params; `UnpackHighlightScreen` creates the actual DB row once the user commits to keeping photos — `file:88-119`.

## Screenshot
_Not yet captured. TODO: capture light + dark for each of the 4 inner steps, plus the Path-B variant of step 1 (with the "N moments found" context line and without the alt-path link)._

## Open questions / follow-ups
- This entire "Unpack A Trip" flow (5 screens) is not mentioned anywhere in the PRD or the January-2026 context doc — it's new since those were written. Worth a PRD addendum describing it as a distinct Retrospective-Traveler flow, since it's substantial (a 4-stage wizard) and not a minor feature.
- Path B's entry is unusual: it's reached only via a `CommonActions.reset` dispatched from `UnpackProcessingScreen`, not a normal `navigate` call — meaning this screen's nav history gets rewritten mid-flow rather than pushed onto normally. Worth keeping in mind if analytics/screen-tracking assumes a simple push stack.
