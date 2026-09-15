# Journal Detail

**Route name:** `JournalDetail`
**File:** `src/screens/JournalDetailScreen.tsx`
**Type:** Stack screen (`presentation: 'card'`)

## Purpose
Full-screen read view for a complete journal/insight entry — date, reverse-geocoded location, and the journal's full text on a paper-like card. The Reflective Traveler persona's primary "revisit what I wrote" screen.

## Entry points & navigation
- Entry: from `EntryDetailScreen`'s branch-journal stat cell or picker modal, from the Home feed's journal cards, from Review — anywhere a journal is listed.
- `route.params`: `{ journalId: string }`.
- Exit / further navigation:
  - Back arrow → `navigation.goBack()`.
  - 3-dot menu → **Edit**: currently just shows an `Alert.alert('Not available', 'Journal editing will be available in a future update.')` — not implemented. **Delete**: confirm `Alert`, then `deleteJournal(journalId)` + `goBack()`.

## Layout anatomy
1. Fixed header: back arrow (left) + 3-dot menu button (right), absolutely positioned over the scroll content.
2. Scrollable content: a single `contentCard` (surface-colored, rounded) containing:
   - Formatted date ("Month D, YYYY", Georgia on iOS / Playfair Display on Android).
   - Location line (only if reverse-geocoding resolved a city/region/country) — same serif style as the date.
   - Full journal `content` text, same serif family, left as one continuous block (no title field rendered separately here).
3. 3-dot dropdown menu (Edit / Delete) — same open/close animation and structure as `EntryDetailScreen`'s menu.

## States
- **`!journal`** (not found/deleted) → header (back button only, no menu) + "Journal not found" centered text.
- **Location resolving**: `locationName` starts empty and is populated asynchronously via reverse geocoding; if the journal has no `latitude`/`longitude`, or geocoding fails, the location line is simply omitted (no error shown, no placeholder).
- **Menu open/closing**: identical two-state (`menuVisible`/`menuExiting`) animated pattern to `EntryDetailScreen`.
- **Edit tapped**: always shows a "not available" alert — this is a stubbed action, not a real edit flow.

## Data & context dependencies
- `useAppContext()` → `journals`, `deleteJournal`.
- `useTheme()` → `colors`.
- Dynamic `import('expo-location')` → `reverseGeocodeAsync({ latitude, longitude })`, building a "City, Region, Country" string from whichever address parts resolve (each part optional).

## Interactions
- Menu overlay: `Animated.parallel` fade (`menuOverlayOpacity`) + scale (`menuDropdownScale`, 0.92→1), open 200/220ms, close 150/150ms with `Easing.in`/`Easing.out` — byte-for-byte the same animation constants as `EntryDetailScreen`'s menu (worth reusing as a shared hook/component if a third screen ever needs the same menu).
- No other custom animation or gesture handling — this is otherwise a static read view.

## Theming notes
Standard `useTheme()` token usage (`colors.background`, `colors.surface`, `colors.textPrimary`, `colors.textSecondary`). The serif font choice (`Georgia` on iOS, `FONTS.playfairDisplay` on Android) for date/location/content is a deliberate "handwritten journal" feel distinct from the app's usual Inter body text — worth preserving if this screen is ever redesigned, since it's one of very few places serif is used for body copy rather than just headlines.

## Reference UI skeleton
Small enough to skeleton almost completely — menu animation timings and delete/edit handlers reduced to comments:

```tsx
// source: src/screens/JournalDetailScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const JournalDetailScreen = () => {
  const { journalId } = route.params;
  const journal = journals.find(j => j.id === journalId);
  const [locationName, setLocationName] = useState('');
  const [menuVisible, setMenuVisible] = useState(false);
  const [menuExiting, setMenuExiting] = useState(false);

  // reverse-geocode journal.latitude/longitude → "City, Region, Country" — :105-135
  // menu open/close Animated.parallel (opacity + scale, 200/220ms open, 150/150ms close) — :138-182
  // handleEdit → Alert.alert('Not available', ...) — :184-188 (stubbed, not a real edit flow)
  // handleDelete → confirm Alert → deleteJournal(journalId) + goBack() — :190-212

  if (!journal) {
    return (
      <View style={styles.container}>
        <View style={styles.header}>
          <TouchableOpacity style={[styles.backButton, { top: actionButtonTop, left: 25 * scaleFactor }]} onPress={() => navigation.goBack()}>
            <BackArrow color={colors.textPrimary} />
          </TouchableOpacity>
        </View>
        <View style={styles.errorContainer}>
          <Text style={styles.errorText}>Journal not found</Text>
        </View>
      </View>
    );
  }

  const formattedDate = new Date(journal.timestamp).toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' });

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <TouchableOpacity style={[styles.backButton, { top: actionButtonTop, left: 25 * scaleFactor }]} onPress={() => navigation.goBack()}>
          <BackArrow color={colors.textPrimary} />
        </TouchableOpacity>
        <TouchableOpacity style={[styles.menuButton, { top: actionButtonTop, right: 25 * scaleFactor }]} onPress={toggleMenu}>
          <Text style={styles.menuButtonText}>...</Text>
        </TouchableOpacity>
      </View>

      {(menuVisible || menuExiting) && (
        <Animated.View style={[styles.menuOverlay, { opacity: menuOverlayOpacity }]} pointerEvents="box-none">
          <TouchableWithoutFeedback onPress={() => closeMenuAnimated()}><View style={StyleSheet.absoluteFill} /></TouchableWithoutFeedback>
          <Animated.View style={[styles.menuDropdown, { top: menuDropdownTop, right: 25 * scaleFactor, transform: [{ scale: menuDropdownScale }] }]}>
            <TouchableOpacity style={styles.menuOption} onPress={handleEdit}>
              <Ionicons name="settings" size={20} color={colors.textPrimary} />
              <Text style={styles.menuOptionText}>Edit</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.menuOption} onPress={handleDelete}>
              <Ionicons name="trash-outline" size={20} color="#DC2626" />
              <Text style={[styles.menuOptionText, styles.menuOptionTextDanger]}>Delete</Text>
            </TouchableOpacity>
          </Animated.View>
        </Animated.View>
      )}

      <ScrollView style={styles.scrollView} contentContainerStyle={[styles.scrollContent, { paddingTop: topBarTop + 60, paddingBottom: insets.bottom + 24 }]}>
        <View style={styles.contentCard}>
          <Text style={styles.date}>{formattedDate}</Text>
          {locationName.trim().length > 0 && <Text style={styles.location}>{locationName}</Text>}
          <Text style={styles.content}>{journal.content}</Text>
        </View>
      </ScrollView>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: colors.background },
  contentCard: { backgroundColor: colors.surface, borderRadius: DIMENSIONS.cardBorderRadius, padding: 24 },
  date: { fontSize: 17, color: colors.textSecondary, fontFamily: Platform.OS === 'ios' ? 'Georgia' : FONTS.playfairDisplay, marginBottom: 1, lineHeight: 22 },
  location: { fontSize: 17, color: colors.textSecondary, fontFamily: Platform.OS === 'ios' ? 'Georgia' : FONTS.playfairDisplay, marginBottom: 6, lineHeight: 22 },
  content: { fontSize: 15, color: colors.textPrimary, fontFamily: Platform.OS === 'ios' ? 'Georgia' : FONTS.playfairDisplay, lineHeight: 23 },
  menuDropdown: { position: 'absolute', backgroundColor: colors.surface, borderRadius: 8, minWidth: 120, paddingVertical: 4, shadowColor: '#000', shadowOpacity: 0.3, shadowRadius: 4, elevation: 8 },
});
```

**Key technique(s):** the platform-conditional serif font (`Georgia` on iOS, `FONTS.playfairDisplay` on Android) applied to all three text elements (`:77-97`) is a deliberate "handwritten journal" feel distinct from the app's usual Inter body text — one of very few places serif is used for body copy rather than just headlines. Worth preserving if this screen is ever redesigned.

## Screenshot
_Not yet captured. TODO: capture light + dark: journal with location resolved, journal with no location, "Journal not found" error state, 3-dot menu open._

## Open questions / follow-ups
- "Edit" in the 3-dot menu is a dead end (`Alert.alert('Not available', ...)`) — confirm with product whether journal editing is genuinely planned (in which case this doc's Entry-points section will need updating once it's wired up) or whether the menu option should be removed/hidden until it exists, since right now it's discoverable but non-functional.
- This screen and `EntryDetailScreen` duplicate the exact same 3-dot menu animation code (same constants, same two-state pattern) — a shared `useAnimatedActionMenu()` hook or `<ActionMenu>` component would remove that duplication; noting for an engineer, not doing it here since this pass is documentation-only.