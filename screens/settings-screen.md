# Settings Screen

**Route name:** `Settings`
**File:** `src/screens/SettingsScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`, `headerShown: false`)

## Purpose
Grouped-list app settings: appearance (theme), Travel Mode toggle, Home map visibility, username, and a destructive full-app-data reset. Standard iOS-Settings-style layout (section headers + rounded-rect rows), not travel-content-specific.

## Entry points & navigation
- Reached from `ReviewScreen`'s settings-gear button (top-right of the segment row) → `navigation.navigate('Settings')`. No other entry point in the codebase.
- No route params.
- Back: circular chevron button → `navigation.goBack()`.
- "Restart app" (after confirmation) → `resetApp()` then a full `CommonActions.reset({ index: 0, routes: [{ name: 'MainTabs' }] })` — a hard reset back to the tab root, not a `goBack()`.
- Dev-only row "Preview onboarding" → `navigation.navigate('OnboardingSlides')` (see Open questions).

## Layout anatomy
Single `ScrollView`, top to bottom:
1. Header: circular back button + "Settings" title, bottom border.
2. **APPEARANCE** section: one row, "Theme" label + current value ("Dark"/"Light") + a `Switch`.
3. **TRAVEL** section: one row, "Travel Mode" + on/off value + `Switch` (green track when on, `#34C759`, distinct from the other switches' theme-driven track color).
4. **HOME** section: one row, "Show map" + Visible/Hidden + `Switch`.
5. **ACCOUNT** section: one row, "Username" + current name (or "Set name" placeholder) + chevron — opens the username modal.
6. Standalone destructive button, "Restart app" (red text, `#c62828`, shows a spinner mid-reset).
7. **DEV** section (explicitly marked in a code comment "DEV — remove before TestFlight", line 407): "Preview onboarding" row.
8. A modal (native `Modal`, `transparent`, `fade`) for editing the username: backdrop (tap to dismiss) → centered card with title, `AppTextInput`, and a "Save" button (spinner while saving).

Every settings section here is a single-row "block" (`rowFirst + rowLast + rowSingle` all applied together) — the multi-row-group styles (`row`, `rowFirst`, `rowLast` used independently) exist in the stylesheet but aren't currently exercised, since every section in this screen happens to have exactly one row.

## States
- **Theme switch**: reflects `isDark` directly (no explicit "System" option — matches `CLAUDE.md`'s note that `ThemeContext` is user-only, no system-preference sync); `onValueChange` calls `setColorScheme('dark' | 'light')`.
- **Travel Mode toggle — turning off requires confirmation**: turning it *on* calls `setTravelMode(true)` immediately; turning it *off* while currently on shows a confirmation `Alert` ("Turn off Travel Mode? Are you sure...") before calling `setTravelMode(false, true)` (lines 89–108) — the asymmetry (confirm only on disable) is deliberate.
- **Username modal**: `usernameModalVisible`; input seeded from current `userName` on open (`openUsernameModal`, lines 54–57); saving is a no-op skip if the trimmed value is unchanged (line 66); `savingName` disables the input and swaps the button for a spinner; save failure shows a generic `Alert.alert('Error', ...)` and leaves the modal open for retry.
- **Restart/reset**: two-step — confirmation `Alert` (destructive style, explicit "This cannot be undone" copy) → `resetting` boolean shows an inline spinner in place of the button label during `resetApp()`; on failure, resets `resetting` to false and shows an error `Alert`, leaving the user on Settings.
- **Username display**: falls back to literal "Set name" placeholder text when `userName` is empty/whitespace-only (`userName?.trim() || 'Set name'`, line 385).

## Data & context dependencies
- `useAppContext()`: `userName`, `setUserName`, `resetApp`, `travelModeEnabled`, `setTravelMode`, `showHomeMap`, `setShowHomeMap`.
- `useTheme()`: `colors`, `isDark`, `setColorScheme`.
- Local components: `AppText`, `AppTextInput` (from `components/common/`), `LoadingSpinner`, `MeshGradientBackground`.

## Interactions
- All three feature toggles are plain `Switch` components — no custom slider/animation.
- Username modal uses `Pressable` backdrop-tap-to-dismiss with `e.stopPropagation()` on the card itself to prevent the backdrop handler from firing when tapping inside the card (line 432).
- `KeyboardAvoidingView` (`behavior: 'padding'` on iOS) wraps just the modal card's contents, not the whole screen.

## Theming notes
- Fully `useTheme().colors`-driven except two literals: the destructive red `#c62828` (`DESTRUCTIVE_COLOR` constant) and Travel Mode's "on" track color `#34C759` (iOS system green) — both intentionally *not* theme tokens, presumably to keep "danger" and "this feature is active" universally recognizable regardless of light/dark.

## Reference UI skeleton
Every section below reuses one "block" pattern: a `sectionHeader` label + a `sectionBlock` wrapper containing rows that combine `rowFirst + rowLast + rowSingle` (all three at once, since every section here happens to have exactly one row — the plain `row` style for multi-row groups exists but is unused).

```tsx
// source: src/screens/SettingsScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
return (
  <View style={styles.container}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
    <View style={styles.header}>
      <TouchableOpacity style={styles.backButton} onPress={() => navigation.goBack()}>
        <Ionicons name="chevron-back" size={24} color={colors.textPrimary} />
      </TouchableOpacity>
      <Text style={styles.headerTitle}>Settings</Text>
    </View>

    <ScrollView style={styles.scroll} contentContainerStyle={styles.scrollContent}>
      <Text style={styles.sectionHeader}>APPEARANCE</Text>
      <View style={styles.sectionBlock}>
        <View style={[styles.row, styles.rowFirst, styles.rowLast, styles.rowSingle]}>
          <View><Text style={styles.rowLabel}>Theme</Text><Text style={styles.rowValue}>{isDark ? 'Dark' : 'Light'}</Text></View>
          <Switch value={isDark} onValueChange={handleAppearanceChange} trackColor={{ false: colors.border, true: colors.borderLight }} thumbColor={colors.white} />
        </View>
      </View>

      <Text style={styles.sectionHeader}>TRAVEL</Text>
      <View style={styles.sectionBlock}>
        <View style={[styles.row, styles.rowFirst, styles.rowLast, styles.rowSingle]}>
          <View><Text style={styles.rowLabel}>Travel Mode</Text><Text style={styles.rowValue}>{travelModeEnabled ? 'On' : 'Off'}</Text></View>
          <Switch value={travelModeEnabled} onValueChange={handleTravelModeToggle} trackColor={{ false: colors.border, true: '#34C759' }} thumbColor={colors.white} />
          {/* '#34C759' iOS system green — the one switch whose "on" track color isn't theme-driven */}
        </View>
      </View>

      <Text style={styles.sectionHeader}>HOME</Text>
      <View style={styles.sectionBlock}>
        <View style={[styles.row, styles.rowFirst, styles.rowLast, styles.rowSingle]}>
          <View><Text style={styles.rowLabel}>Show map</Text><Text style={styles.rowValue}>{showHomeMap ? 'Visible' : 'Hidden'}</Text></View>
          <Switch value={showHomeMap} onValueChange={setShowHomeMap} trackColor={{ false: colors.border, true: colors.borderLight }} thumbColor={colors.white} />
        </View>
      </View>

      <Text style={styles.sectionHeader}>ACCOUNT</Text>
      <View style={styles.sectionBlock}>
        <TouchableOpacity style={[styles.row, styles.rowFirst, styles.rowLast, styles.rowSingle]} onPress={openUsernameModal}>
          <Text style={styles.rowLabel}>Username</Text>
          <View style={{ flexDirection: 'row', alignItems: 'center' }}>
            <Text style={styles.rowValue} numberOfLines={1}>{userName?.trim() || 'Set name'}</Text>
            <Ionicons name="chevron-forward" size={20} color={colors.textSecondary} />
          </View>
        </TouchableOpacity>
      </View>

      <TouchableOpacity style={styles.rowDestructive} onPress={handleRestart} disabled={resetting}>
        {resetting ? <LoadingSpinner size="small" color={DESTRUCTIVE_COLOR} /> : <AppText style={styles.rowDestructiveText}>Restart app</AppText>}
      </TouchableOpacity>

      {/* DEV — remove before TestFlight (line 407) */}
      <Text style={styles.sectionHeader}>DEV</Text>
      <View style={styles.sectionBlock}>
        <TouchableOpacity style={[styles.row, styles.rowFirst, styles.rowLast, styles.rowSingle]} onPress={() => navigation.navigate('OnboardingSlides')}>
          <Text style={styles.rowLabel}>Preview onboarding</Text>
          <Ionicons name="chevron-forward" size={20} color={colors.textSecondary} />
        </TouchableOpacity>
      </View>
    </ScrollView>

    <Modal visible={usernameModalVisible} transparent animationType="fade" onRequestClose={closeUsernameModal} statusBarTranslucent>
      <Pressable style={styles.modalBackdrop} onPress={closeUsernameModal}>
        <Pressable style={styles.modalCard} onPress={(e) => e.stopPropagation()}>
          <KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
            <Text style={styles.modalTitle}>Username</Text>
            <AppTextInput style={styles.modalInput} value={usernameInput} onChangeText={setUsernameInput} placeholder="Your name" editable={!savingName} />
            <TouchableOpacity style={styles.modalButton} onPress={handleSaveUsername} disabled={savingName}>
              {savingName ? <LoadingSpinner size="small" /> : <Text style={styles.modalButtonText}>Save</Text>}
            </TouchableOpacity>
          </KeyboardAvoidingView>
        </Pressable>
      </Pressable>
    </Modal>
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.background },
  header: { flexDirection: 'row', alignItems: 'center', paddingHorizontal: SPACING.md, paddingTop: insets.top + SPACING.sm, paddingBottom: SPACING.md, borderBottomWidth: 1, borderBottomColor: colors.border },
  backButton: { width: 40, height: 40, borderRadius: 20, backgroundColor: colors.surfaceVariant, borderWidth: 1, borderColor: colors.border, alignItems: 'center', justifyContent: 'center', marginRight: SPACING.sm },
  sectionHeader: { fontSize: 12, fontWeight: FONT_WEIGHTS.semibold, color: colors.textSecondary, marginTop: SPACING.xl, marginBottom: SPACING.sm, marginHorizontal: SPACING.lg, letterSpacing: 0.5 },
  row: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', paddingVertical: SPACING.md, paddingHorizontal: SPACING.lg, backgroundColor: colors.surface, borderTopWidth: 1, borderTopColor: colors.border },
  rowSingle: { borderRadius: 12, borderWidth: 1, borderColor: colors.border },
  rowLabel: { fontSize: 16, color: colors.textPrimary, flex: 1 },
  rowValue: { fontSize: 14, color: colors.textSecondary, marginRight: SPACING.sm },
  rowDestructive: { backgroundColor: colors.surface, borderWidth: 1, borderColor: colors.border, borderRadius: 12, marginHorizontal: SPACING.lg, marginTop: SPACING.lg, paddingVertical: SPACING.md, alignItems: 'center' },
  rowDestructiveText: { fontSize: 16, color: DESTRUCTIVE_COLOR },  // '#c62828'
  sectionBlock: { marginHorizontal: SPACING.lg, borderRadius: 12, overflow: 'hidden', borderWidth: 1, borderColor: colors.border },
  modalBackdrop: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: SPACING.lg, backgroundColor: 'rgba(0,0,0,0.4)' },
  modalCard: { width: '100%', maxWidth: 360, backgroundColor: colors.surface, borderRadius: 16, borderWidth: 1, borderColor: colors.border, padding: SPACING.xl },
});
```

**Key technique(s):**
- The asymmetric confirm-only-on-disable pattern for Travel Mode (`src/screens/SettingsScreen.tsx:89-108`) is a small but easy-to-miss UX rule worth preserving if this toggle is ever reimplemented:
```tsx
const handleTravelModeToggle = useCallback((value: boolean) => {
  if (!value && travelModeEnabled) {
    Alert.alert('Turn off Travel Mode?', 'Are you sure you want to turn it off?', [
      { text: 'Cancel', style: 'cancel' },
      { text: 'Turn Off', onPress: () => setTravelMode(false, true) },
    ]);
  } else {
    setTravelMode(value);
  }
}, [travelModeEnabled, setTravelMode]);
```

## Screenshot
_Not yet captured. TODO: capture light + dark for: the main list, the username modal open, and the restart confirmation `Alert`._

## Open questions / follow-ups
- **The DEV section is still present and explicitly self-flagged for removal** ("DEV — remove before TestFlight", line 407) — this is a real pre-release housekeeping item, not just a documentation gap. Surface to the team: either remove it or gate it behind a dev-only build flag before the next TestFlight upload (see `journeyon/TESTFLIGHT_CHECKLIST.md` in `CLAUDE.md`'s pre-release process).
- No entries here for notifications, account/auth (sign-in, data export), or privacy — consistent with `PRD_JourneyOn_MVP.md`'s "Out of Scope" list (no auth beyond local-first for MVP), so likely intentional rather than missing, but worth a designer sanity-check once real backend auth (`TR4`) lands.
