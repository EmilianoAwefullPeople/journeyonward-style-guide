# Trip Album Create / Edit Screen

**Route name:** `TripAlbumCreate` (create mode) and `TripAlbumEdit` (edit mode) — both routes map to the same component, `TripAlbumCreateScreen`
**File:** `src/screens/TripAlbumCreateScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
A 4-step wizard for creating a new trip album or editing an existing one. Serves the Retrospective and Live-Capture travelers when they want to give their trip a name and date range so entries/journals can be organized under it.

## Entry points & navigation
- **Create mode** (`TripAlbumCreate`, no `albumId` param): reached from the tab bar's center "+" plus-menu → "New Trip Album" (`handlePlusMenuAction('TripAlbumCreate')` in `AppNavigator.tsx`), and also used mid-onboarding when `pendingOnboardingPath === 'trip_album'`.
- **Edit mode** (`TripAlbumEdit`, `route.params.albumId` set): reached from `TripAlbumDetailScreen`'s edit action.
- Route param: `{ albumId?: string }` — presence of `albumId` alone determines `isEditMode` (`const isEditMode = Boolean(albumId)`, line 46); the route name itself (`TripAlbumEdit` vs `TripAlbumCreate`) is not read in code.
- **Exit on create success**: dispatches a full navigation `CommonActions.reset` to `MainTabs` → `Review` tab with `params: { initialSection: 'tripAlbums' }` (lines 154–170) — this is a hard reset, not a simple `goBack()`, so the create flow always lands the user on Review's Trip Albums section afterward.
- **Exit on edit success**: simple `navigation.goBack()`.
- Back arrow (step 1) or "← Back" text (steps 2–4) either steps back one wizard step or calls `navigation.goBack()` from step 1.

## Layout anatomy
1. `MeshGradientBackground` full-bleed behind everything.
2. Header row: back/close control only (no title text in the header itself).
3. `ScrollView` containing exactly one of 4 step bodies:
   - **Step 1 — Title**: italic serif label "Give your album a title", a borderless `TextInput` for the name, "Next →" button (only rendered once title is non-empty).
   - **Step 2 — Start date**: label + native `DateTimePicker` (`mode="date"`, iOS spinner), "Next →" button (always enabled).
   - **Step 3 — End date**: label + `DateTimePicker` with `minimumDate={startDate}`, "Next →" button.
   - **Step 4 — Description**: label "Extra thoughts?", multiline `TextInput` (2000 char max), primary button labeled "Save" or "Skip & Save" depending on whether description is empty.
4. No progress indicator (dots/bar) for the 4 steps — step position is only implied by which fields are on screen.

## States
- **Create vs. edit**: `isEditMode` (line 46) changes the button label logic (edit mode always shows "Save"; create mode shows "Skip & Save" when description is empty) and the post-save navigation (reset-to-Review vs. `goBack()`).
- **Prefill on edit**: `useEffect` (lines 65–72) seeds `title`/`startDate`/`endDate`/`description` from `existingAlbum` found via `tripAlbums.find((a) => a.id === albumId)`.
- **Step 1 gating**: "Next →" is not rendered at all (not just disabled) until `title.trim().length > 0` (`canProceedStep1`, line 183).
- **End-date clamp**: a `useEffect` (lines 75–79) forces `endDate` back to `startDate` whenever the user picks a start date later than the current end date.
- **Saving**: `saving` boolean disables the step-4 button and guards `handleSave` against double-taps (line 123); no visible spinner is shown during the save.
- **Error**: on a thrown error from `createTripAlbum`/`updateTripAlbum`, shows a native `Alert.alert('Error', ...)` with a mode-specific message (lines 172–177); the form state is preserved so the user can retry.
- **Keyboard-open**: `isKeyboardOpen`/`keyboardHeight` (from `keyboardWillShow`/`keyboardWillHide` on iOS) pad the scroll content bottom so the active input and button aren't hidden (lines 82–99, 184).

## Data & context dependencies
- `useAppContext()`: `createTripAlbum`, `updateTripAlbum`, `tripAlbums` (to look up `existingAlbum`), `pendingOnboardingPath`, `completeOnboarding`.
- `useTheme()`: `colors`, `isDark` (passed to `MeshGradientBackground`).
- `useSafeAreaInsets()` for header top padding.
- No location, weather, or media services touched by this screen at all.

## Interactions
- Native `DateTimePicker` spinner wheels for both dates (no custom slider/graphic).
- `TouchableWithoutFeedback` wraps steps 1 and 4 to dismiss the keyboard on background tap.
- `KeyboardAvoidingView` (`behavior: 'padding'` on iOS) wraps the whole screen.
- No swipe gesture between steps — navigation between steps is button-only.

## Theming notes
- All colors come from `useTheme().colors` (no hardcoded light/dark branches); `isDark` is only used to flip `MeshGradientBackground`'s `lightMode` prop.
- Title/description labels use `FONTS.playfairDisplay` italic bold — the only screen-specific typographic flourish; body/input text uses `FONTS.inter`.

## Reference UI skeleton

```tsx
// source: src/screens/TripAlbumCreateScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
return (
  <KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'} style={styles.container}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

    <View style={[styles.header, { paddingTop: insets.top + SPACING.sm }]}>
      <TouchableOpacity onPress={handleBack} style={styles.backButton} accessibilityLabel={step === 1 ? 'Close' : 'Back'}>
        {step === 1
          ? <BackArrow width={19} height={21} color={colors.textPrimary} />
          : <Text style={styles.backButtonText}>← Back</Text>}
      </TouchableOpacity>
    </View>

    <ScrollView keyboardShouldPersistTaps="handled" contentContainerStyle={[styles.scrollContent, { paddingBottom: contentBottomPadding }]}>
      {step === 1 && (
        <TouchableWithoutFeedback onPress={handleDismissKeyboard}>
          <View style={styles.stepContent}>
            <Text style={styles.label}>Give your album a title</Text>
            <TextInput
              style={styles.titleInput}
              placeholder="Album title"
              placeholderTextColor={colors.textSecondary}
              value={title}
              onChangeText={setTitle}
              autoCapitalize="words"
              maxLength={120}
            />
            {canProceedStep1 && (  // title.trim().length > 0 — button doesn't render at all otherwise, line 265
              <TouchableOpacity style={styles.primaryButton} onPress={handleNext}>
                <Text style={styles.primaryButtonText}>Next →</Text>
              </TouchableOpacity>
            )}
          </View>
        </TouchableWithoutFeedback>
      )}

      {step === 2 && (
        <View style={styles.stepContent}>
          <Text style={styles.label}>Date of start of the trip</Text>
          <View style={styles.pickerContainer}>
            <DateTimePicker
              value={startDate}
              mode="date"
              display={Platform.OS === 'ios' ? 'spinner' : 'default'}
              onChange={(_, date) => date && setStartDate(date)}  // triggers end-date clamp useEffect, lines 75-79
              textColor={colors.textPrimary}
            />
          </View>
          <TouchableOpacity style={styles.primaryButton} onPress={handleNext}>
            <Text style={styles.primaryButtonText}>Next →</Text>
          </TouchableOpacity>
        </View>
      )}

      {step === 3 && (
        <View style={styles.stepContent}>
          <Text style={styles.label}>Date of end of the trip</Text>
          <View style={styles.pickerContainer}>
            <DateTimePicker
              value={endDate}
              mode="date"
              minimumDate={startDate}  // can never pick an end before start
              display={Platform.OS === 'ios' ? 'spinner' : 'default'}
              onChange={(_, date) => date && setEndDate(date)}
              textColor={colors.textPrimary}
            />
          </View>
          <TouchableOpacity style={styles.primaryButton} onPress={handleNext}>
            <Text style={styles.primaryButtonText}>Next →</Text>
          </TouchableOpacity>
        </View>
      )}

      {step === 4 && (
        <TouchableWithoutFeedback onPress={handleDismissKeyboard}>
          <View style={[styles.stepContent, { paddingBottom: contentBottomPadding }]}>
            <Text style={styles.descriptionTitle}>Extra thoughts?</Text>
            <TextInput
              style={styles.descriptionInput}  // minHeight: 120, multiline, textAlignVertical: 'top'
              placeholder="Write a description..."
              value={description}
              onChangeText={setDescription}
              multiline
              maxLength={2000}
            />
            <TouchableOpacity style={styles.primaryButton} onPress={handleSave} disabled={saving}>
              <Text style={styles.primaryButtonText}>
                {isEditMode ? 'Save' : description.trim().length > 0 ? 'Save' : 'Skip & Save'}
              </Text>
            </TouchableOpacity>
          </View>
        </TouchableWithoutFeedback>
      )}
    </ScrollView>
  </KeyboardAvoidingView>
);

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.background },
  header: { paddingHorizontal: SPACING.md, paddingBottom: SPACING.sm },
  backButton: { alignSelf: 'flex-start', paddingVertical: SPACING.sm, paddingRight: SPACING.md },
  backButtonText: { fontFamily: FONTS.inter, fontSize: 17, fontWeight: FONT_WEIGHTS.semibold, color: colors.textPrimary },
  scrollContent: { flexGrow: 1, paddingHorizontal: SPACING.lg, paddingTop: SPACING.lg },
  stepContent: { flex: 1 },
  label: {  // shared by step 1's label and step 4's descriptionTitle
    fontFamily: FONTS.playfairDisplay, fontSize: 22, fontStyle: 'italic',
    fontWeight: FONT_WEIGHTS.bold, color: colors.textPrimary, marginBottom: SPACING.lg,
  },
  titleInput: {
    fontFamily: FONTS.inter, fontSize: 18, color: colors.textPrimary,
    backgroundColor: 'transparent', paddingHorizontal: 0, paddingVertical: SPACING.lg, marginBottom: SPACING.xl,
  },
  pickerContainer: { marginBottom: SPACING.xl, alignItems: 'center' },
  descriptionInput: {
    fontFamily: FONTS.inter, fontSize: 16, color: colors.textPrimary,
    backgroundColor: 'transparent', paddingVertical: SPACING.lg, minHeight: 120, marginBottom: SPACING.xl,
  },
  primaryButton: {
    backgroundColor: colors.surface, borderRadius: DIMENSIONS.borderRadiusModal,
    paddingVertical: Math.round(SPACING.md * 0.85), paddingHorizontal: Math.round(SPACING.xl * 0.85),
    alignItems: 'center', alignSelf: 'flex-end',
  },
  primaryButtonText: { fontFamily: FONTS.inter, fontSize: 14, fontWeight: FONT_WEIGHTS.semibold, color: colors.textPrimary },
});
```

**Key technique(s):**
- No step-progress dots/bar exists anywhere in the tree above — step position is inferred purely from which of the 4 `{step === N && (...)}` blocks is mounted (`src/screens/TripAlbumCreateScreen.tsx:365-368`).
- The full-stack reset after a successful create is worth calling out on its own — it's a `CommonActions.reset`, not `goBack()`, deliberately routing the user into Review's Trip Albums section to see the new album (`src/screens/TripAlbumCreateScreen.tsx:154-170`):
```tsx
navigation.dispatch(CommonActions.reset({
  index: 0,
  routes: [{ name: 'MainTabs', state: { index: 1, routes: [
    { name: 'Home' },
    { name: 'Review', params: { initialSection: 'tripAlbums' } },
  ] } }],
}));
```

## Screenshot
_Not yet captured. TODO: capture light + dark for each of the 4 steps, plus the edit-mode prefilled state._

## Open questions / follow-ups
- **PRD mismatch**: `PRD_JourneyOn_MVP.md` Flow 3 describes trip album creation as including a **Location** field and a full pre-trip reflection step (chatbot or form collecting Plans/Intentions/Travel Reasons). None of that exists in this screen — it's strictly title → start date → end date → description. `TripAlbum` model fields like `locationName`, `plans`, `intentions`, `travelReasons` are only preserved pass-through on edit (spread from `existingAlbum`, lines 131–141), never captured or edited here. Confirm with product whether pre-trip reflection was intentionally cut for MVP or still needs a home (possibly folded into `TripAlbumDetailScreen` post-creation?).
- No step-progress UI (e.g. "1 of 4" or dots) — worth a screenshot-driven design check on whether users get lost mid-wizard.
