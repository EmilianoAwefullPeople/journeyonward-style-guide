# Saved Postcards Screen

**Route name:** `SavedPostcards`
**File:** `src/screens/SavedPostcardsScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
A simple gallery of every postcard the user has saved (via the bookmark action in `PostcardEditorScreen`), so they can reopen one to keep editing or re-export. Serves the Travel Sharer persona's "look back at what I've made" need.

## Entry points & navigation
- Registered in `AppNavigator.tsx` as `SavedPostcards`, but **no screen in the current codebase navigates to it** (`grep` for `navigate('SavedPostcards'` returns nothing) — it's reachable only by deep link or by code not yet written. Presumably intended to be linked from `SettingsScreen` or a "Your Postcards" entry point in Review, but that wiring doesn't exist yet.
- No route params.
- Tapping a postcard → `navigation.navigate('PostcardEditor', { postcardId: item.id })`.
- Exits via back chevron (`navigation.goBack()`).

## Layout anatomy
1. Header: back chevron + "Saved Postcards" title.
2. Two-column `FlatList` grid of `PostcardThumbnail` components (from `src/components/postcards/`), each wrapped in a drop-shadowed `TouchableOpacity`.
3. Empty state (replaces the grid entirely): bookmark-outline icon, "No saved postcards" title, one-line instructions.

## States
- **Empty**: `postcards.length === 0` → centered empty state with instructions to use the "+" button and save from the editor.
- **Populated**: two-column grid, each cell sized to `(screenWidth - padding*2 - gap) / 2`.

## Data & context dependencies
- `useAppContext()` → `postcards` (read-only; no create/update/delete happens on this screen).
- `useTheme()` → `colors`.
- `useSafeAreaInsets()`.
- Renders via `PostcardThumbnail`, a shared component that must independently render each template's saved photo/label state as a static thumbnail (not the live editable template component).

## Interactions
- Tap a thumbnail → opens it in `PostcardEditor` for further editing/export. No swipe-to-delete or long-press menu exists on this screen.

## Theming notes
- Standard `useTheme()` usage for background/title/empty-state text; the shadow style (`postcardShadow`) is fixed (not theme-varied), consistent with postcard thumbnails always appearing on a physical-object-like white/shadowed card regardless of app theme.

## Reference UI skeleton
Whole render tree, business logic stripped. `GRID_GAP = 12`. `CARD_WIDTH = (Dimensions.get('window').width - SPACING.lg*2 - GRID_GAP) / 2` (computed once at module scope — see "Key technique").

```tsx
// source: src/screens/SavedPostcardsScreen.tsx (skeleton — business logic stripped)
return (
  <View style={styles.container}> {/* flex:1, bg: colors.background */}
    <View style={styles.header}> {/* row, paddingTop: insets.top+8, paddingHorizontal: SPACING.lg, paddingBottom:16 */}
      <TouchableOpacity style={styles.backButton} onPress={() => navigation.goBack()}>
        <Ionicons name="chevron-back" size={24} color={colors.textPrimary} />
      </TouchableOpacity>
      <AppText style={styles.title}>Saved Postcards</AppText> {/* 20px bold, flex:1 */}
    </View>

    {postcards.length === 0 ? (
      <View style={styles.empty}> {/* flex:1, center, gap:12, paddingBottom:80 */}
        <Ionicons name="bookmark-outline" size={48} color={colors.textSecondary} />
        <AppText style={styles.emptyTitle}>No saved postcards</AppText> {/* 17px semibold */}
        <AppText style={styles.emptySubtitle}> {/* 14px, secondary, centered, paddingHorizontal:40 */}
          Create a postcard using the + button and tap Save to store it here.
        </AppText>
      </View>
    ) : (
      <FlatList
        data={postcards}
        keyExtractor={(p) => p.id}
        numColumns={2}
        columnWrapperStyle={{ gap: GRID_GAP }}
        contentContainerStyle={styles.list} // paddingHorizontal: SPACING.lg
        renderItem={({ item }) => (
          <TouchableOpacity
            style={styles.item} // width: CARD_WIDTH, marginBottom: GRID_GAP, borderRadius:12
            onPress={() => navigation.navigate('PostcardEditor', { postcardId: item.id })}
          >
            <View style={styles.postcardShadow}> {/* fixed drop-shadow, not theme-varied */}
              <PostcardThumbnail postcard={item} width={CARD_WIDTH} />
            </View>
          </TouchableOpacity>
        )}
      />
    )}
  </View>
);
```

**Key technique(s):**
- No pull-to-refresh, no loading state, no pagination — `postcards` is assumed to already be a fully-loaded in-memory array from `AppContext`.
- `CARD_WIDTH` is computed once from `Dimensions.get('window').width` at module scope rather than `useWindowDimensions()` inside the component (unlike every other postcard screen) — won't respond to a runtime dimension change — `file:25`.

## Screenshot
_Not yet captured. TODO: capture light + dark for both the empty state and a populated 2-column grid — the grid needs seeded postcard data first, since no in-app flow currently reaches this screen to create that data via the UI._

## Open questions / follow-ups
- **This screen currently has no entry point anywhere in the app's navigation.** That's the single most important finding for this doc: either wiring is missing (e.g. a Settings row or Review tab entry), or the screen is legacy/aspirational and not yet slated for this release. Worth confirming with product/design before spending screenshot time on it.
- `CARD_WIDTH` is computed once from `Dimensions.get('window').width` at module scope rather than `useWindowDimensions()` inside the component (unlike every other postcard screen) — won't respond to runtime dimension changes (e.g. rotation, if ever supported). Minor inconsistency worth normalizing if this screen gets finished.
