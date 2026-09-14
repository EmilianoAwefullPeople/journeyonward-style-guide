# Effects & Atmosphere

> Reference only. See repo [README](../README.md). Snippets below are illustrative — stripped of app-specific imports/wiring — not drop-in modules.

This is the most distinctive visual signature of v1: the **"Organic Mesh"** aesthetic — layered, blurred-feeling color gradients standing in for a true radial/mesh gradient (which React Native doesn't support natively), topped with subtle grain and used behind loading screens, cards without photos, and atmospheric moments.

## 1. Mesh gradient background

**Problem it solves**: React Native has no native radial or true mesh gradient. The look is faked by stacking several `LinearGradient` layers at different angles and opacities so colors bleed into each other like a radial gradient would.

**Recipe**:
1. Solid base `View` in a dark base color.
2. 3–4 diagonal `LinearGradient` "dark-shading" layers (black → transparent) at varying angles/opacities — these add depth/vignetting, not color.
3. Up to 3 "accent blobs" — each an accent color faded to transparent, positioned at different corners (top-right, bottom-left, center-left) — these are the actual color identity of the screen and are passed in per-context (e.g. a specific trip's mood colors).
4. A grain overlay on top (see below) so the gradient doesn't read as flat/plasticky.

```tsx
// Illustrative — not the real component
<View style={styles.container}>
  <View style={[styles.base, { backgroundColor: baseColor }]} />

  {/* dark-shading layers for depth, e.g.: */}
  <LinearGradient
    colors={['rgba(0,0,0,0.6)', 'rgba(0,0,0,0)']}
    locations={[0, 0.7]}
    start={{ x: 0, y: 0 }} end={{ x: 0.8, y: 0.8 }}
    style={StyleSheet.absoluteFill}
  />
  {/* ...2-3 more dark-shading layers at other angles */}

  {/* accent blob, top-right, faded to transparent */}
  <LinearGradient
    colors={[withOpacity(accentColor, 0.22), withOpacity(accentColor, 0.13), 'transparent']}
    locations={[0, 0.35, 1]}
    start={{ x: 1, y: 0 }} end={{ x: 0.15, y: 0.65 }}
    style={StyleSheet.absoluteFill}
  />

  <GrainOverlay />
</View>
```

Key numbers from the source: accent blob peak opacity ~0.22, mid ~0.13 — accents are meant to be a *wash*, not a dominant color block. Dark mode uses stronger dark-shading opacities (0.3–0.6) than light mode (0.06–0.12).

**Where it's used**: loading screen, feed-card placeholders for photo-less entries (video/audio/missing media — so the card never reads as "empty"), atmospheric moments tied to a trip's color mood.

## 2. Grain overlay

A near-invisible noise texture laid over mesh backgrounds so they read as "tactile" rather than flat/digital. v1's actual implementation is a pragmatic fallback: a single semi-transparent white `View` at ~4% opacity (`rgba(255,255,255,0.04)`), because true SVG `feTurbulence` noise wasn't reliably supported on native at the time. The design intent (documented directly in source) was true noise texture; the shipped version is the flat-opacity approximation.

**Worth deciding explicitly in v2.0**: if the platform/library now supports real noise/turbulence textures (e.g. via Skia, a noise image asset, or a shader), it may be worth doing properly rather than re-shipping the flat-opacity approximation.

## 3. Gradient text

Text filled with a gradient (not a solid color) via a mask: the gradient renders behind an invisible copy of the text used purely as a stencil.

```tsx
// Illustrative
<MaskedView
  maskElement={<Text style={style}>{children}</Text>}
>
  <LinearGradient colors={[babyBlue, royalGold, fern]} start={{x:0,y:0}} end={{x:1,y:0}}>
    <Text style={[style, { opacity: 0 }]}>{children}</Text>
  </LinearGradient>
</MaskedView>
```

Default gradient stops: `babyBlue → royalGold → fern`, left to right — the same three colors used in the tab-bar active indicator (see navigation doc), so gradient text reads as "the same accent system," not a one-off effect. Used specifically to mark **bookmarked** entry titles — the gradient is a status signal (this is saved/favorited), not decoration applied everywhere.

## Design rules distilled from the source system

These read like intentional constraints, worth keeping as *rules* even if implementation changes:

- No solid strokes on mesh surfaces — edges should be soft/glowing, not hard-bordered.
- Shadows are long and colored (a dark slate tone at high transparency), never pure black.
- Accent/highlight colors act as "light leaks" — always low-opacity top layers, never a flat fill.
- Every mesh surface gets a grain pass — the intent was to avoid anything reading as a flat, glossy digital gradient.
