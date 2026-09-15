# Onboarding — Slides

**Route name:** `OnboardingSlides`
**File:** `src/screens/onboarding/OnboardingSlidesScreen.tsx`
**Type:** Stack screen (root of the stack when onboarding isn't complete)

## Purpose
The app's first-run pitch: four horizontally-paged, full-bleed slides that sell the product story (messy camera roll → organized memories → shareable recaps/postcards → the emotional/human layer) before the user picks a starting path. Serves all personas equally since it's pre-account, pre-data.

## Entry points & navigation
- Entry: `AppNavigator`'s root stack sets `initialRouteName` to `'OnboardingSlides'` whenever `hasCompletedOnboarding` (from `AppContext`) is false — it's the very first screen a new install sees, before `MainTabs` even mounts.
- No `route.params`.
- Exit: tapping "Next" on the last slide, or "Skip" at any point, calls `navigation.navigate('OnboardingPath')`. `gestureEnabled: false` and `animation: 'fade'` are set on this stack screen (in `AppNavigator.tsx`), so there's no swipe-back — the only way out is forward.

## Layout anatomy
Top to bottom, per slide (all absolutely stacked, full-screen):
1. `SlideBackground` — solid base color + two `LinearGradient` "blobs" + a bottom legibility scrim, unique per slide (blue → green → gold → plum).
2. A per-slide `renderVisual` graphic, either full-bleed (slide 1 only) or centered in the top ~50% of the screen (slides 2–4).
3. Bottom-anchored text block: `headline` (Playfair Display, 28px) + `body` (Inter, 15px, translucent white).
4. `GrainOverlay` — texture layer painted over every slide.
5. "Skip" text button, top-right, above the grain.
6. Bottom controls bar: pagination dots + a white pill CTA button ("Next" with an arrow icon, or "Get started" on the last slide).

The whole thing is one horizontal, paging `FlatList` over 4 `SlideItem`s; only `Skip`, the dots, and the CTA sit outside the FlatList as fixed overlays.

## States
- **Slide 1 — "messy":** `MessyGridVisual`, a full-bleed 4-column photo grid (`assets/onboarding/camera-roll/`) that auto-scrolls vertically, alternating direction per column, faded at top/bottom into the slide background.
- **Slide 2 — "organized":** `OrganizedCardsVisual`, one of 5 destination photo sets (central card + two rotated secondary tiles peeking behind it) cross-fading every ~3.6s; the first set only starts revealing once `isActive` (i.e. the user has actually swiped to this slide) turns true.
- **Slide 3 — "loop":** `RecapShareVisual`, a `PeekCarousel` of 4 real postcard images auto-advancing every ~3.2s.
- **Slide 4 — "payoff":** `EmotionVisual`, 4 "moment cards" (photo + emotion-note caption pill) cross-fading every ~3.2s.
- **Reduce Motion on:** every auto-animation short-circuits to its resting/first-frame state (`presences[0].setValue(1)`, scroll columns don't loop) via `useReduceMotion()`.
- **Last slide:** CTA label switches from "Next" (with arrow icon) to "Get started" (no icon); the forward-arrow icon is conditionally omitted (`{!isLast && <Ionicons .../>}`).
- Dots: active dot is a wider gold pill (20×6), inactive dots are small translucent-white circles (6×6); tapping any dot jumps `FlatList` to that slide via `scrollToIndex`.

## Data & context dependencies
- No `AppContext`/`ThemeContext` reads — this screen is intentionally theme-independent (always dark, brand-colored backgrounds regardless of system/user theme).
- `useReduceMotion()` hook (`src/hooks/useReduceMotion.ts`) gates every animation loop.
- `useSafeAreaInsets()` / `useWindowDimensions()` for layout math.
- Local `useNavigation<NativeStackNavigationProp<RootStackParamList>>()` only.

## Interactions
- Horizontal paging `FlatList` (`pagingEnabled`, `bounces={false}`) drives `currentIndex` via `onMomentumScrollEnd` (rounds `contentOffset.x / width`).
- All slide visuals use the legacy `Animated` API (not Reanimated) — `Animated.loop`, `Animated.sequence`, `Animated.parallel`, all `useNativeDriver: true`.
- `ScrollingPhotoColumn` (slide 1): infinite vertical loop built from two back-to-back copies of the same photo set, so the loop reset lands on a visually identical frame (seamless wrap) instead of snapping.
- `PeekCarousel` (slide 3, and reused by slide 4's card layout conceptually): prepends the last card and appends the first so a neighbor is always visible even on first paint, before the loop ever wraps.
- Tap targets: Skip button, 4 dot buttons, CTA button — all `TouchableOpacity` with `accessibilityRole="button"` and explicit `accessibilityLabel`s.

## Theming notes
Deliberately not theme-aware — every color is a literal (`'#FFFFFF'`, `rgba(...)`) or a brand token (`COLORS.babyBlue`, `COLORS.royalGold`, `COLORS.fern`), not `useTheme()`. The root `View`'s background is hardcoded to `SLIDE_THEMES[0].base` to avoid a flash before the FlatList's first slide paints.

## Reference UI skeleton
Top-level structure — one `SlideItem` per slide inside a paging `FlatList`, plus the fixed Skip/dots/CTA overlay chrome. Each slide's own visual component (`MessyGridVisual`, `OrganizedCardsVisual`, `RecapShareVisual`, `EmotionVisual`) is collapsed to a one-line placeholder here since they're four structurally-independent subtrees — see their own `file:line` for internals:

```tsx
// source: src/screens/onboarding/OnboardingSlidesScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const SLIDE_THEMES = [ /* 4 entries: base color + 2 gradient-blob configs each — see :64-113 */ ];

// Layered backdrop shared by every slide: solid base, two color-blob gradients, one
// bottom legibility scrim (see :115-138) — full, real implementation, small enough to keep as-is:
const SlideBackground = ({ theme }) => (
  <>
    <View style={[StyleSheet.absoluteFillObject, { backgroundColor: theme.base }]} />
    <LinearGradient colors={[hex(theme.blob1Color, theme.blob1Alpha), 'transparent']} start={theme.blob1Start} end={theme.blob1End} style={StyleSheet.absoluteFillObject} />
    <LinearGradient colors={['transparent', hex(theme.blob2Color, theme.blob2Alpha)]} start={theme.blob2Start} end={theme.blob2End} style={StyleSheet.absoluteFillObject} />
    <LinearGradient colors={['transparent', 'rgba(0,0,0,0.6)']} start={{ x: 0.5, y: 0.32 }} end={{ x: 0.5, y: 1 }} style={StyleSheet.absoluteFillObject} />
  </>
);

const SLIDES = [
  { key: 'messy', headline: 'Hundreds of pictures... \nmany stories left untold', theme: SLIDE_THEMES[0], renderVisual: () => <MessyGridVisual />, fullBleedVisual: true },
  { key: 'organized', headline: 'Memories organized, \nmoments to be relived', theme: SLIDE_THEMES[1], renderVisual: (isActive) => <OrganizedCardsVisual isActive={isActive} /> },
  { key: 'loop', headline: 'Meaningful stories, \nready for sharing & reliving', theme: SLIDE_THEMES[2], renderVisual: () => <RecapShareVisual /> },
  { key: 'payoff', headline: 'The story is more \nthan the photos', theme: SLIDE_THEMES[3], renderVisual: () => <EmotionVisual /> },
]; // see :913-943 for full headline/body copy

// One slide's full render tree (see :958-1001) — visualHeight = 50% of slide height:
const SlideItem = ({ slide, width, height, topPad, bottomPad, isActive }) => (
  <View style={{ width, height, overflow: 'hidden' }}>
    <SlideBackground theme={slide.theme} />
    {slide.fullBleedVisual ? (
      <View style={StyleSheet.absoluteFillObject} pointerEvents="none">{slide.renderVisual(isActive)}</View>
    ) : (
      <View style={{ position: 'absolute', top: topPad, left: 0, right: 0, height: height * 0.5, alignItems: 'center', justifyContent: 'center' }} pointerEvents="none">
        {slide.renderVisual(isActive)}
      </View>
    )}
    <View style={{ position: 'absolute', left: 0, right: 0, bottom: bottomPad, paddingHorizontal: SPACING.lg, gap: 16 }}>
      <Text style={sStyles.headline}>{slide.headline}</Text>
      {slide.body ? <Text style={sStyles.body}>{slide.body}</Text> : null}
    </View>
  </View>
);

const OnboardingSlidesScreen = () => {
  const { width, height } = useWindowDimensions();
  const insets = useSafeAreaInsets();
  const [currentIndex, setCurrentIndex] = useState(0);
  const topPad = insets.top + 52;
  const bottomPad = insets.bottom + CONTROLS_HEIGHT + 24; // CONTROLS_HEIGHT = 144

  // goToSlide / handleNext / handleSkip / onMomentumScrollEnd / getItemLayout — see :1024-1055

  const isLast = currentIndex === SLIDES.length - 1;

  return (
    <View style={[styles.root, { backgroundColor: SLIDE_THEMES[0].base }]}>
      <FlatList
        data={SLIDES}
        horizontal
        pagingEnabled
        bounces={false}
        showsHorizontalScrollIndicator={false}
        extraData={currentIndex}
        renderItem={({ item, index }) => (
          <SlideItem slide={item} width={width} height={height} topPad={topPad} bottomPad={bottomPad} isActive={index === currentIndex} />
        )}
      />

      <GrainOverlay />

      <TouchableOpacity style={[styles.skip, { top: insets.top + 16 }]} onPress={handleSkip} activeOpacity={0.7}>
        <Text style={styles.skipText}>Skip</Text>
      </TouchableOpacity>

      <View style={[styles.controls, { paddingBottom: insets.bottom + 32 }]}>
        <View style={styles.dots}>
          {SLIDES.map((_, i) => (
            <TouchableOpacity key={i} onPress={() => goToSlide(i)}>
              <View style={[styles.dot, i === currentIndex ? styles.dotActive : styles.dotInactive]} />
            </TouchableOpacity>
          ))}
        </View>
        <TouchableOpacity style={styles.cta} onPress={handleNext} activeOpacity={0.85}>
          <Text style={styles.ctaLabel}>{isLast ? 'Get started' : 'Next'}</Text>
          {!isLast && <Ionicons name="arrow-forward" size={16} color="#111111" />}
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  root: { flex: 1 },
  skip: { position: 'absolute', right: SPACING.lg, zIndex: 20 },
  skipText: { fontFamily: FONTS.inter, fontSize: 15, color: 'rgba(255,255,255,0.65)' },
  controls: { position: 'absolute', left: 0, right: 0, bottom: 0, zIndex: 20, paddingHorizontal: SPACING.lg, gap: 18, alignItems: 'center' },
  dots: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  dot: { borderRadius: 100 },
  dotActive: { width: 20, height: 6, backgroundColor: COLORS.royalGold },
  dotInactive: { width: 6, height: 6, backgroundColor: 'rgba(255,255,255,0.35)' },
  cta: { flexDirection: 'row', alignItems: 'center', justifyContent: 'center', backgroundColor: '#FFFFFF', borderRadius: 100, paddingVertical: 16, alignSelf: 'stretch', gap: 8 },
  ctaLabel: { fontFamily: FONTS.inter, fontSize: 16, fontWeight: FONT_WEIGHTS.semibold, color: '#111111' },
});

// Slide text styles (sStyles) — headline: Playfair 28/35, bold, white; body: Inter 15/22, white @ 75% (see :1199-1215)
```

**Key technique(s):**
- The gold-pill dot (`dotActive`, 20×6 vs. `dotInactive` 6×6, `:1170-1179`) is the same "wide pill = active" convention used for the tab-bar indicator in `patterns/navigation-and-tab-bar.md`.
- `OrganizedCardsVisual` (`:538-608`, not expanded above) gives every carousel set its own permanent `Animated.Value` ("presence") rather than reusing two current/next slots — reused/reassigned slots raced between a JS-side prop swap and a native-driven animation reset, causing a one-frame flash of the wrong set at full opacity; a stable value-per-set removes the reassignment entirely. Worth this pattern before rebuilding any crossfading carousel elsewhere in the app.

## Screenshot
_Not yet captured. TODO: capture all 4 slides in sequence (dark, no light variant applicable), plus the Reduce-Motion-on resting state of slide 2._

## Open questions / follow-ups
- The comment block at the top of the file (lines 1–10) documents the intended visual style closely enough that a screenshot mainly serves to confirm timing/feel (auto-advance pacing, crossfade smoothness) rather than layout — worth capturing as a short screen recording rather than a static PNG if design review needs to judge pacing.
