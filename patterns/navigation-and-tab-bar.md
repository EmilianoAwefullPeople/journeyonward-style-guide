# Navigation & Tab Bar

> Reference only. See repo [README](../README.md). Snippets are illustrative.

## Structure

Three main tabs — **Studio | Home | Review** — plus a center **capture** action, in a custom tab bar (not the default React Navigation tab bar chrome).

## Active-tab indicator: animated gradient underline

Instead of a solid-color underline or tinted icon, the active tab gets a thin (2pt) gradient underline that animates in on tab change:

- Gradient: `babyBlue → royalGold → fern`, horizontal (left to right) — the same three-stop accent gradient used in `GradientText` elsewhere, so the whole app's "accent" language stays consistent.
- Positioned just under the tab label (`bottom: -12` relative to label, not flush at the very bottom of the bar).
- Animates via `scaleX` from 0→1 (not width, not opacity) over 240ms with an ease-out cubic curve — reads as the underline "drawing itself in" under the newly active tab.
- Respects **Reduce Motion**: when enabled, the indicator snaps to full width instantly instead of animating.

```tsx
// Illustrative
const progress = useRef(new Animated.Value(0)).current;
useEffect(() => {
  if (reduceMotion) { progress.setValue(1); return; }
  progress.setValue(0);
  Animated.timing(progress, { toValue: 1, duration: 240, easing: Easing.out(Easing.cubic), useNativeDriver: true }).start();
}, [activeTab]);

<View style={{ width, height: 2, borderRadius: 1, overflow: 'hidden' }}>
  <Animated.View style={{ width, transform: [{ scaleX: progress }] }}>
    <LinearGradient colors={[babyBlue, royalGold, fern]} start={{x:0,y:0}} end={{x:1,y:0}} style={{ flex: 1 }} />
  </Animated.View>
</View>
```

## Capture button

Center tab-bar action rendered as a **gradient-bordered pill**, not a flat circle: an outer gradient-filled wrapper (76×50, 25 radius) with 2px padding, containing an inner solid-filled pill (23 radius) — creates a thin gradient ring effect around the button rather than a gradient fill or a flat accent color.

## The gradient border as a general "featured" signal

The same `babyBlue → royalGold → fern` gradient-ring treatment used on the capture button shows up again elsewhere in the app as a card border — on Journey Recap cards and on a promo card ("Unpack A Trip") in the capture sheet (see `screens/capture-action-sheet.png`, `screens/review-trips-dark.png`). In both cases it's marking "this is a special/featured item," not just tab-bar or button chrome.

**Worth generalizing as a reusable primitive**: a `GradientBorder` wrapper (outer gradient-filled container + 2px inset solid-fill inner content) that any card or button can opt into when it needs to read as elevated/featured — rather than reimplementing the outer-wrapper/inner-pill nesting per use site the way v1 did.

## Layout math worth keeping as a pattern

v1 centralizes tab-bar height + safe-area math into a single small module that every scrollable screen imports, rather than each screen re-deriving "how much bottom padding do I need to clear the tab bar":

- `getMainTabBarRowHeight()` — label row height only.
- `getMainTabBarChromeHeight(insetsBottom)` — row height + `max(safe-area inset, 20pt minimum gutter)`.
- `getEffectiveMainTabInset(insetsBottom, visibilityProgress)` — accounts for the tab bar animating in/out (e.g. hiding on scroll), so scroll-content padding can track the bar's *current* height, not just its resting height.

This "one source of truth for bottom chrome height" pattern is worth replicating regardless of what navigation library v2.0 uses — it's what prevented content from being clipped behind the tab bar on different device sizes (notch vs. no notch, SE vs. Pro Max) as the bar's own layout changed over time.
