# Postcard Templates Screen

**Route name:** `PostcardTemplates`
**File:** `src/screens/PostcardTemplatesScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
Lets the Travel Sharer persona pick a starting point for a postcard: either a curated "Templates" layout (single/multi-photo compositions with a fixed visual treatment — camera-viewfinder chrome, handwritten note overlay, location tag, etc.) or a plain "Gallery" of the user's own photos to hand straight to the (stubbed) Story editor. This is the entry screen of the postcard-creation flow.

## Entry points & navigation
- Entered from the global "+" action sheet in `AppNavigator.tsx` (`handlePlusMenuAction('PostcardTemplates')`, "Create A Postcard" grid item).
- No route params.
- Tapping a **Templates** or **Collages** card navigates to `PostcardEditor` with `{ templateId: template.id }`.
- Tapping a **Gallery** photo navigates to `PostcardStoryEditor` with `{ photoUri, entryId }`.
- Back chevron calls `navigation.goBack()`.

## Layout anatomy
1. Header: back chevron + "New Postcard" title.
2. `TabPill` — animated two-segment pill switch ("Templates" / "Gallery"), sliding indicator.
3. **Templates tab** (default):
   - "Collages" section label + horizontal `ScrollView` of small collage cards (portrait-ish ratio, unlabeled, from `TEMPLATE_REGISTRY` filtered to `category === 'collage'`).
   - "Templates" section label + vertical 2-column grid (rows built manually via `renderRows`) of styled-template cards (`category === 'styled'`), each `cardWidth × cardWidth*4/3`.
4. **Gallery tab**: 3-column `FlatList` of the user's own photos (`entries` flattened over `photoPaths`, drafts excluded, newest first) or an empty state.

## States
- **Templates tab, populated**: always populated (registry is static).
- **Gallery tab, empty**: `galleryPhotos.length === 0` → centered `images-outline` icon + "No photos yet".
- **Gallery tab, populated**: 3-column photo grid.
- Tab switch itself is a state (`activeTab: 'templates' | 'gallery'`) driving which body renders.

## Data & context dependencies
- `useAppContext()` → `entries` (to build the Gallery tab's photo list).
- `useTheme()` → `colors`.
- `useSafeAreaInsets()`, `useWindowDimensions()` for responsive card sizing.
- `useReduceMotion()` hook (inside `TabPill`) to skip the sliding-indicator animation.
- Static `TEMPLATE_REGISTRY` from `src/config/templateRegistry` — the catalog of template/collage ids, categories, and display names.
- Local `MOCK_PHOTOS` map of bundled sample images (`assets/postcard-mockups/*`) used only to render template previews in the picker — not real user data.

## Interactions
- `TabPill`: tap a segment to switch tabs; the active-segment indicator slides via `Animated.timing` (`PILL_SLIDE_MS = 240ms`, `Easing.out(Easing.cubic)`), animating `left`/`width` (not `useNativeDriver`, since layout properties are animated). Segment positions are measured via `onLayout` and cached in a ref; first measurement snaps instantly, subsequent activations animate.
- Template/collage/gallery cards: `TouchableOpacity` with `activeOpacity` feedback, no gestures beyond tap.

## Theming notes
- Card chrome (`colors.surfaceVariant`, `colors.border`) is theme-aware via `useTheme()`.
- Template *preview content* itself is intentionally NOT theme-aware — each `TemplatePreview` case hard-codes colors like `#000`, `rgba(0,0,0,0.55)` to match how the real template renders against a photo background regardless of app theme.
- `TabPill`'s pill/indicator background switches between `rgba(255,255,255,0.08)` (dark) and `rgba(0,0,0,0.06)` (light) via `isDark`.

## Reference UI skeleton
Whole render tree, business logic stripped. `COLUMNS = 2`, `CARD_GAP = 12`, `GALLERY_COLUMNS = 3`, `GALLERY_CELL_GAP = 2`, `PILL_SLIDE_MS = 240`. `cardWidth = (screenWidth - SPACING.lg*2 - CARD_GAP) / COLUMNS`, `previewHeight = cardWidth * 4/3`, `collageCardWidth = cardWidth * 0.72`, `collageCardHeight = collageCardWidth * 1.6`, `gallerySize = (screenWidth - GALLERY_CELL_GAP*(GALLERY_COLUMNS+1)) / GALLERY_COLUMNS`.

```tsx
// source: src/screens/PostcardTemplatesScreen.tsx (skeleton — business logic stripped)

// ── TabPill sub-component (lines 557-711) ──────────────────────────────────
// Animates an absolute-positioned indicator's left/width (not scale/opacity)
// to slide between two arbitrarily-sized segment labels. See "Key technique".
const TabPill = ({ activeTab, onChange }) => (
  <View style={styles.outer}> {/* alignItems:'center', paddingBottom:16 */}
    <View style={styles.pill}> {/* row, bg: isDark 'rgba(255,255,255,.08)' : 'rgba(0,0,0,.06)', borderRadius:22, padding:4, border:colors.border */}
      {indicatorReady && (
        <Animated.View // absolute, top/bottom:4, borderRadius:18, bg: isDark 'rgba(255,255,255,.16)' : '#fff'
          style={[styles.indicator, { width: indicatorW, transform: [{ translateX: indicatorX }] }]}
        />
      )}
      {TABS.map(tab => ( // [{key:'templates'}, {key:'gallery'}]
        <TouchableOpacity
          key={tab.key}
          onLayout={/* measures x/width into layoutsRef, drives indicator anim — see excerpt */}
          onPress={() => tab.key !== activeTab && onChange(tab.key)}
        >
          <AppText style={[styles.segmentText, isActive && styles.segmentTextActive]}>{tab.label}</AppText>
        </TouchableOpacity>
      ))}
    </View>
  </View>
);

// ── TemplatePreview sub-component (lines 114-545) ───────────────────────────
// Hand-drawn mock rendering per templateId, using bundled MOCK_PHOTOS — NOT the
// real editable template components. Representative case shown; T-01, T-03,
// T-04, T-05, T-06, T-07, T-11, C-01..C-05, T-12 each have their own case with
// a distinct hand-laid-out View tree (2x2 grids, camera-HUD chrome, chat
// bubbles, calendar strips, etc.) — see the file directly for each.
const TemplatePreview = ({ templateId, width, height }) => {
  switch (templateId) {
    case 'T-02': // full-bleed camera-viewfinder chrome
      return (
        <View style={{ width, height, backgroundColor: '#000' }}>
          <Image source={MOCK_PHOTOS.t02[0]} style={{ width, height }} resizeMode="cover" />
          <View style={topBarStyle} pointerEvents="none">{/* mono HUD text + temp chip */}</View>
          {/* 4x corner brackets, absolutely positioned */}
          <View style={bottomStripStyle} pointerEvents="none">{/* frame#, date/time, location, watermark */}</View>
        </View>
      );
    default:
      return <View style={{ width, height, padding: 8, gap: 4 }}>{/* two flex:1 dark rects */}</View>;
  }
};

// ── Screen ───────────────────────────────────────────────────────────────────
return (
  <View style={styles.container}> {/* flex:1, bg: colors.background */}
    <View style={styles.header}> {/* row, paddingTop: insets.top+8, paddingHorizontal: SPACING.lg, paddingBottom:16 */}
      <TouchableOpacity style={styles.backButton} onPress={() => navigation.goBack()}>
        <Ionicons name="chevron-back" size={24} color={colors.textPrimary} />
      </TouchableOpacity>
      <AppText style={styles.title}>New Postcard</AppText> {/* 20px bold */}
    </View>

    <TabPill activeTab={activeTab} onChange={setActiveTab} />

    {activeTab === 'templates' ? (
      <ScrollView contentContainerStyle={styles.list}> {/* padding: SPACING.lg */}
        <AppText style={styles.sectionLabel}>Collages</AppText>
        <ScrollView horizontal contentContainerStyle={styles.collageRow}> {/* gap: CARD_GAP */}
          {collageTemplates.map(t => (
            <TouchableOpacity key={t.id} style={styles.collageCard} onPress={() => navigation.navigate('PostcardEditor', { templateId: t.id })}>
              <TemplatePreview templateId={t.id} width={collageCardWidth} height={collageCardHeight} />
            </TouchableOpacity>
          ))}
        </ScrollView>

        <AppText style={[styles.sectionLabel, styles.templatesLabel]}>Templates</AppText> {/* marginTop:28 */}
        {/* renderRows: chunks styledTemplates into rows of COLUMNS */}
        {rowsOf(styledTemplates, COLUMNS).map((row, i) => (
          <View key={i} style={styles.row}> {/* row, gap: CARD_GAP, marginBottom: CARD_GAP */}
            {row.map(t => (
              <TouchableOpacity key={t.id} style={styles.card} onPress={() => navigation.navigate('PostcardEditor', { templateId: t.id })}>
                {/* card: width:cardWidth, borderRadius:12, bg:colors.surfaceVariant, border:colors.border */}
                <View style={styles.previewArea}> {/* height: previewHeight, bg:'#111' */}
                  <TemplatePreview templateId={t.id} width={cardWidth} height={previewHeight} />
                </View>
              </TouchableOpacity>
            ))}
          </View>
        ))}
      </ScrollView>
    ) : galleryPhotos.length === 0 ? (
      <View style={styles.emptyGallery}> {/* flex:1, center, gap:12 */}
        <Ionicons name="images-outline" size={48} color={colors.textSecondary} />
        <AppText style={styles.emptyGalleryText}>No photos yet</AppText>
      </View>
    ) : (
      <FlatList
        data={galleryPhotos}
        numColumns={GALLERY_COLUMNS}
        contentContainerStyle={styles.galleryGrid} // padding: GALLERY_CELL_GAP
        renderItem={({ item }) => (
          <TouchableOpacity style={styles.galleryCell} onPress={() => navigation.navigate('PostcardStoryEditor', { photoUri: item.uri, entryId: item.entry.id })}>
            <Image source={{ uri: item.uri }} style={{ width: gallerySize, height: gallerySize }} resizeMode="cover" />
          </TouchableOpacity>
        )}
      />
    )}
  </View>
);
```

**Key technique(s):**
- The animated segmented-tab indicator measures each segment's layout on mount (`onLayout` → `layoutsRef`), then animates `left`/`width` via `Animated.parallel` (not scale/opacity, since it must slide between arbitrarily-sized labels) — `file:612-638`.
- Template preview cells never use `flex:1` or `StyleSheet.absoluteFill` on an `Image`; every cell computes an explicit pixel `width`/`height` up front and passes it straight to `Image`, because the flex/absoluteFill combination left cropped photos rendering as an ambiguous solid sliver — `file:100-111`.

## Screenshot
_Not yet captured. TODO: capture light + dark for: Templates tab (default), Collages horizontal row, Gallery tab populated, Gallery tab empty state._

## Open questions / follow-ups
- The PRD/context docs (pre-2026-09) don't mention postcards at all — this entire feature (templates, editor, saved postcards, story editor) appears to postdate the PRD and isn't reflected there. Worth a PRD addendum if postcards are staying in scope.
- `TemplatePreview`'s per-template mock rendering (T-01 … T-12, C-01 … C-05) duplicates layout logic that presumably also lives in the real template components under `src/components/postcards/templates/`; a future refactor could risk the two drifting out of sync since the preview is hand-maintained separately (explicitly noted in the file's own comments, e.g. line 37-39).
