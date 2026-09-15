# Postcard Story Editor Screen

**Route name:** `PostcardStoryEditor`
**File:** `src/screens/PostcardStoryEditorScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
Explicitly a **placeholder**. Confirms the photo the user picked from `PostcardTemplatesScreen`'s Gallery tab and tells them the real Instagram-Stories-style graphic editor (drawing, text, stickers) isn't built yet. Exists purely so the Gallery tab's tap target has somewhere to go without dead-ending silently.

## Entry points & navigation
- From `PostcardTemplatesScreen`'s Gallery tab → `navigation.navigate('PostcardStoryEditor', { photoUri, entryId })`.
- Params: `{ photoUri: string; entryId?: string }` — `entryId` is accepted but unused in this screen's body.
- Exits via back chevron (`navigation.goBack()`) only. No save/continue action exists.

## Layout anatomy
1. Full-bleed `Image` of the selected photo (`StyleSheet.absoluteFillObject`).
2. Dark scrim overlay (`rgba(0,0,0,0.45)`), centered content: sparkles icon, "Story editor coming soon" title, one-line explanatory subtitle.
3. Back chevron, absolutely positioned top-left.

## States
This screen has exactly one state — there's no data loading, no error path, no empty state. It always renders the same placeholder content over whichever photo was passed in.

## Data & context dependencies
- None beyond `route.params.photoUri`. Does not read `AppContext`, `ThemeContext`, or any service.

## Interactions
- Back chevron only. No gestures, no other taps.

## Theming notes
- Not theme-aware at all — hardcoded `#000` background and white text, matching the fixed dark "full-bleed photo" convention also used by `PostcardEditorScreen`'s overlays.

## Reference UI skeleton
The entire render tree — worth citing in full since the whole screen is this short, and is the clearest signal that this is an intentional stub, not an oversight. No sub-components, no conditional branches.

```tsx
// source: src/screens/PostcardStoryEditorScreen.tsx (skeleton — identical to the real file, it's this short)
return (
  <View style={styles.container}> {/* flex:1, bg:'#000' — hardcoded, not theme-aware */}
    <Image
      source={{ uri: photoUri }}
      style={styles.photo} // ...StyleSheet.absoluteFillObject
      resizeMode="cover"
    />
    <View style={styles.overlay}> {/* ...absoluteFillObject, bg:'rgba(0,0,0,0.45)', center, gap:12, paddingHorizontal: SPACING.xl */}
      <Ionicons name="sparkles-outline" size={40} color="#fff" />
      <AppText style={styles.title}>Story editor coming soon</AppText> {/* 20px bold, white, centered */}
      <AppText style={styles.subtitle}>
        Drawing, text, and stickers for this photo are on the way.
      </AppText> {/* 14px, rgba(255,255,255,0.8), centered */}
    </View>
    <TouchableOpacity
      style={styles.backButton} // absolute, top: insets.top+8, left: SPACING.lg, 40x40, zIndex:1
      onPress={() => navigation.goBack()}
    >
      <Ionicons name="chevron-back" size={24} color="#fff" />
    </TouchableOpacity>
  </View>
);
```

**Key technique(s):** Nothing non-obvious — the entire file is ~90 lines with zero conditional branches, matching the fact that this is a stub the file's own doc-comment describes as "a future project," not partially-built production UI.

## Screenshot
_Not yet captured. TODO: capture one screenshot (theme doesn't matter — screen is hardcoded dark) showing the placeholder over a sample photo._

## Open questions / follow-ups
- This confirms the "Story editor" mentioned only implicitly by the Gallery tab's existence is a known, explicitly-flagged gap (see the file's own doc-comment: "The real editor is a future project"), not something missing by accident. Worth tracking as a real backlog item rather than rediscovering it via code archaeology later.
- The unused `entryId` param suggests the real editor is expected to need it (e.g. to pull the entry's location/date/emotion for auto-generated stickers) — worth keeping the param passthrough intact when this screen is eventually built out.
