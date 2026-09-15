# Onboarding — Path Selection

**Route name:** `OnboardingPath`
**File:** `src/screens/onboarding/OnboardingPathScreen.tsx`
**Type:** Stack screen

## Purpose
Phase 2 of onboarding: lets the new user declare their situation ("I have travel memories" / "I'm on a trip" / "Just looking around") and routes them straight into the matching flow, so the first thing they do in the app is relevant to them rather than a blank Home screen.

## Entry points & navigation
- Entry: from `OnboardingSlidesScreen` via "Next" on the last slide or "Skip" at any point (`navigation.navigate('OnboardingPath')`).
- No `route.params`.
- Exit, per card selected (`handleSelect`):
  - **"I have travel memories"** (`unpack`, featured card) → `setPendingOnboardingPath('unpack')` then `navigation.navigate('UnpackTripDetails')`. Onboarding completion is deferred until that flow finishes.
  - **"I'm on a trip (or planning one)"** (`trip_album`) → same deferred pattern, navigates to `TripAlbumCreate`.
  - **"Just looking around"** (`explore`) → completes onboarding immediately (`await completeOnboarding('explore')`) and resets the nav stack to `MainTabs` (`navigation.reset({ index: 0, routes: [{ name: 'MainTabs' }] })`), so there's no back-stack into onboarding afterward.
- `gestureEnabled: false`, `animation: 'slide_from_right'` (set in `AppNavigator.tsx`) — no swipe-back here either.

## Layout anatomy
1. Small "Journey Onward" brand wordmark (Playfair Display, secondary text color) at the top of a scroll view.
2. Header block: title "Where are you right now?" (Playfair Display, 32px) + subtitle "The right starting point is here." (Inter, 16px).
3. Three option cards, stacked vertically with 12px gaps:
   - **FeaturedCard** ("unpack") — gold gradient border (`COLORS.babyBlue → royalGold → fern`), gold-tinted icon chip.
   - Two **StandardCard**s ("trip_album", "explore") — plain surface background, 1px border, neutral icon chip.
   - Each card: icon chip (46×46, rounded) · headline + subtext text block · trailing chevron.

## States
- No loading/empty/error states — this screen is static, driven entirely by the hardcoded `PATH_OPTIONS` array (3 entries).
- The only branch is `option.featured` (`true` only for `unpack`), which switches which of the two card components renders.
- `handleSelect` is `async` for the `explore` branch only (awaits `completeOnboarding`); the other two branches navigate synchronously after a state write.

## Data & context dependencies
- `useAppContext()` → `completeOnboarding(path)`, `setPendingOnboardingPath(path)`.
- `useTheme()` → `colors.background`, `colors.textPrimary`, `colors.textSecondary`, `colors.surface`, `colors.surfaceVariant`, `colors.border` (all consumed by both card variants — this screen IS theme-aware, unlike the slides screen before it).

## Interactions
- Three `TouchableOpacity` cards (`activeOpacity={0.8}`), each with `accessibilityRole="button"` and an `accessibilityLabel` set to the card's headline.
- No animation on this screen — it's a static list, no `Animated`/Reanimated usage.

## Theming notes
Unlike `OnboardingSlidesScreen`, this screen fully adopts `useTheme()` for backgrounds, text, borders and the standard-card surface — only the featured card's gradient border and icon tint use static brand colors (`COLORS.babyBlue`, `COLORS.royalGold`, `COLORS.fern`), which is deliberate: those are "always-brand" accents, not theme-following surfaces.

## Reference UI skeleton
The whole screen is small enough to skeleton almost verbatim — this IS effectively the real structure, with only `handleSelect`'s branching and the `PATH_OPTIONS` data stripped to comments:

```tsx
// source: src/screens/onboarding/OnboardingPathScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const PATH_OPTIONS = [
  // 'unpack' (featured: true, icon: 'sparkles') → UnpackTripDetails
  // 'trip_album' (featured: false, icon: 'book-outline') → TripAlbumCreate
  // 'explore' (featured: false, icon: 'compass-outline') → MainTabs (see :35-60)
];

const OnboardingPathScreen = () => {
  const { colors } = useTheme();
  const insets = useSafeAreaInsets();

  const handleSelect = async (option) => { /* explore: completeOnboarding + reset; others: setPendingOnboardingPath + navigate — see :68-77 */ };

  return (
    <View style={[styles.container, { backgroundColor: colors.background }]}>
      <ScrollView
        contentContainerStyle={[styles.scroll, { paddingTop: insets.top + 48, paddingBottom: insets.bottom + 32 }]}
        showsVerticalScrollIndicator={false}
      >
        <View style={styles.brandHeader}>
          <Text style={[styles.brandName, { color: colors.textSecondary }]}>Journey Onward</Text>
        </View>

        <View style={styles.header}>
          <Text style={[styles.title, { color: colors.textPrimary }]}>Where are you right now?</Text>
          <Text style={[styles.subtitle, { color: colors.textSecondary }]}>The right starting point is here.</Text>
        </View>

        <View style={styles.cards}>
          {PATH_OPTIONS.map((option) =>
            option.featured
              ? <FeaturedCard key={option.id} option={option} colors={colors} onPress={() => handleSelect(option)} />
              : <StandardCard key={option.id} option={option} colors={colors} onPress={() => handleSelect(option)} />
          )}
        </View>
      </ScrollView>
    </View>
  );
};

// Featured variant: LinearGradient ring (padding: 1.5 leaves the ring visible) wrapping the same
// card content as StandardCard, plus a gold-tinted icon chip (see :119-147).
const FeaturedCard = ({ option, colors, onPress }) => (
  <LinearGradient colors={[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]} start={{ x: 0, y: 0 }} end={{ x: 1, y: 1 }} style={styles.featuredGradientBorder}>
    <TouchableOpacity style={[styles.card, styles.featuredCardInner, { backgroundColor: colors.surfaceVariant }]} onPress={onPress} activeOpacity={0.8}>
      <View style={[styles.iconWrap, { backgroundColor: COLORS.royalGold + '22' }]}>
        <Ionicons name={option.icon} size={22} color={COLORS.royalGold} />
      </View>
      <View style={styles.textBlock}>
        <Text style={[styles.cardHeadline, { color: colors.textPrimary }]}>{option.headline}</Text>
        <Text style={[styles.cardSubtext, { color: colors.textSecondary }]}>{option.subtext}</Text>
      </View>
      <Ionicons name="chevron-forward" size={16} color={colors.textSecondary} />
    </TouchableOpacity>
  </LinearGradient>
);

// Standard variant: plain surface + 1px border, neutral icon chip; identical inner row (see :149-170).
const StandardCard = ({ option, colors, onPress }) => (
  <TouchableOpacity style={[styles.card, { backgroundColor: colors.surface, borderColor: colors.border, borderWidth: 1 }]} onPress={onPress} activeOpacity={0.8}>
    <View style={[styles.iconWrap, { backgroundColor: colors.surfaceVariant }]}>
      <Ionicons name={option.icon} size={22} color={colors.textPrimary} />
    </View>
    <View style={styles.textBlock}>
      <Text style={[styles.cardHeadline, { color: colors.textPrimary }]}>{option.headline}</Text>
      <Text style={[styles.cardSubtext, { color: colors.textSecondary }]}>{option.subtext}</Text>
    </View>
    <Ionicons name="chevron-forward" size={16} color={colors.textSecondary} />
  </TouchableOpacity>
);

const styles = StyleSheet.create({
  container: { flex: 1 },
  scroll: { paddingHorizontal: SPACING.lg },
  brandHeader: { marginBottom: 20 },
  brandName: { fontFamily: FONTS.playfairDisplay, fontSize: 14, letterSpacing: 0.6 },
  header: { marginBottom: 36, gap: 8 },
  title: { fontFamily: FONTS.playfairDisplay, fontSize: 32, lineHeight: 40, letterSpacing: -0.3 },
  subtitle: { fontFamily: FONTS.inter, fontSize: 16, lineHeight: 22 },
  cards: { gap: 12 },
  featuredGradientBorder: { borderRadius: 18, padding: 1.5 }, // the 1.5px padding is what leaves the ring visible
  card: { flexDirection: 'row', alignItems: 'center', borderRadius: 16, paddingHorizontal: SPACING.md, paddingVertical: 18, gap: 14 },
  iconWrap: { width: 46, height: 46, borderRadius: 12, alignItems: 'center', justifyContent: 'center', flexShrink: 0 },
  textBlock: { flex: 1, gap: 4 },
  cardHeadline: { fontFamily: FONTS.inter, fontSize: 16, fontWeight: FONT_WEIGHTS.semibold, lineHeight: 22 },
  cardSubtext: { fontFamily: FONTS.inter, fontSize: 13, lineHeight: 18 },
});
```

**Key technique(s):** the gradient-ring border is a `LinearGradient` sized to the card with 1.5px `padding` (`:207-210`), leaving a colored ring around an inner `TouchableOpacity` that paints the real (opaque) card background — not a border-color trick, since RN borders can't be a gradient directly.

## Screenshot
_Not yet captured. TODO: capture light + dark._

## Open questions / follow-ups
None — this screen's behavior is fully determined by its static option list and is small enough that code reading covers it completely; only visual polish (spacing, color accuracy) needs a screenshot.
