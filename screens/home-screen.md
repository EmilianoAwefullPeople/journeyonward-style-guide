# Home

**Route name:** `Home` (tab, inside the `MainTabs` navigator registered as stack route `MainTabs`)
**File:** `src/screens/HomeScreen.tsx` (~2830 lines — the largest and most stateful screen in the app)
**Type:** Tab screen

## Purpose
The app's default landing screen and primary feed: a mixed stream of the user's entries, journals, and (when eligible) trip Recap/Stories promo cards, plus quick-capture affordances (camera, journal text) and the Travel Mode toggle. Serves the Live-Capture and Retrospective personas most directly — it's where captured content actually surfaces.

## Entry points & navigation
- Entry: initial tab in `MainTabs` (`Tab.Navigator initialRouteName="Home"`); also the `tabPress` listener scrolls to top if already on Home (Instagram/Twitter-style re-tap-to-top).
- No `route.params` — this screen reads everything from `AppContext`.
- Exits:
  - Tapping an entry card → `navigation.navigate('EntryDetail', { entryId })` (deferred via `InteractionManager.runAfterInteractions`).
  - Tapping the journal/pen icon in the floating capture pill → `navigation.navigate('NotesEditor', { journalPlaceholder })`, paired with a local `notesTransitionProgress` Animated value that grows the pill into a full-screen sheet-like transition before the navigation completes.
  - Tapping the camera icon → `navigation.navigate('Capture')`.
  - Both capture actions first call `scrollToTopThen(callback)` — scrolls the feed to the top, waits 80ms, then navigates, so cards aren't visible behind the incoming modal.

## Layout anatomy
Top to bottom (feed mode), inside a `FlashList` with a non-virtualized `ListHeaderComponent`:
1. **Travel Mode glass card** (`BlurView` on iOS, translucent `View` fallback on Android) — title "Travel Mode", "On/Off" state text, and a `Switch`. Rendered only when no trip is currently active (`!tripHeaderData`).
2. **Trip Header card** (replaces the glass card when a trip is active) — same Travel Mode switch, plus "Current Trip: {names}" and "Day X of Y" (or just "Day X" if the trip has no end date).
3. **InteractiveMap** (`src/components/feed/InteractiveMap.tsx`) — collapsible, shown only when `showHomeMap` (an `AppContext` flag) is true; toggled via `mapCollapsed` state.
4. **HomeMemoriesFeedControls** — order toggle (random/chronological) + view toggle (feed/gallery) icons, "Your Memories" header row.
5. **Content**: either the `FlashList` of full-bleed feed cards (feed mode) or `HomeGalleryMasonryGrid` (two-column Pinterest-style masonry, gallery mode) — see States below.
6. **Floating capture pill** — bottom-right, glassmorphic (`BlurView`), containing a pen icon (→ NotesEditor) and a camera icon (→ Capture); only rendered/visible while the main tab bar is hidden (scroll-driven).
7. Long-press overlay (feed & gallery) — a dimming layer + action row (delete, edit, create journal) anchored to the pressed card's measured layout.

## States
- **Loading (initial):** feed empty + `!loadTimedOut` → small `LoadingSpinner` + "Preparing your memories..." italic text. Flips to the empty-state text after 1200ms (`loadTimedOut`) if still empty, rather than spinning forever for genuinely-empty accounts.
- **Empty (feed mode):** centered placeholder text — "Capture Travel Memories To See Them Here!" if a trip is active and not showing all memories, else "Your Photos And Experiences Will Be Shown Here!" (`galleryEmptyPlaceholder` computes the same string for gallery mode).
- **Populated — feed view:** one full-bleed card per item (`renderEntryCard`/`renderJournalCard`/`renderTripStoryCard`), aspect ratio pseudo-randomized per entry (`getFeedAspectRatio`, seeded by `entry.id` + `sessionSeed` so it's stable within a session but reshuffles on pull-to-refresh).
- **Populated — gallery view:** two-column masonry via `HomeGalleryMasonryGrid`, narrower trip-card aspect ratio (1.4 vs. 1.1 in feed).
- **Travel-mode-filtered vs. "show all":** `showAllMemories` toggle scopes the feed to only the active trip's items when a trip is active (auto-resets to `false` whenever `travelModeEnabled` flips true, i.e. every fresh trip start).
- **Refreshing (pull-to-refresh):** `isRefreshing` drives a 4-step sequence — fade cards out (220ms) → reshuffle (`sessionSeed` regenerated, random mode only) + reset scroll/prefetch → fade cards back in (380ms) → spinner dismissed 420ms after fade-out starts.
- **Long-press active on a card:** `longPressActiveEntryId` set, card scaled to 1.05×, dimming overlay fades in (200ms); auto-clears if the entry is deleted elsewhere while the overlay is open.
- **Notes-editor transition:** `isTransitioningToNotes` true → the capture pill area animates (`notesTransitionProgress`, 550ms bezier) into an expanding full-width sheet shape instead of the compact pill; resets instantly (no animation) if the user returns via the `focus` listener mid-transition.
- Entry card variants: has-photo (renders `EntryPhotoImage`) vs. no-photo (`FeedCardPoster` showing a media-type icon placeholder) vs. gallery placeholders for video/audio (`pinterestVideoPlaceholder` / `pinterestAudioPlaceholder`).
- Bookmarked entries (`entry.isBookmarked`) render their title with `GradientText` instead of plain `Text`, in both feed and gallery card variants.

## Data & context dependencies
- `useAppContext()` → `userName`, `entries`, `journals`, `tripAlbums`, `createJournal`, `deleteEntry`, `travelModeEnabled`, `activeTripAlbums`, `setTravelMode`, `showHomeMap`.
- `useTheme()` → `colors`, `isDark`.
- `useMainTabBarVisibility()` → `hidden`, `visibilityProgress`, `reportScrollOffset` — drives whether the tab bar (and thus the floating capture pill's position) is shown/hidden based on scroll direction.
- `useFeedMediaPriority(feedUris, feedVisibleKey)` — image loading priority based on which cards are actually visible (from `onViewableItemsChanged`, debounced 150ms).
- `useSafeAreaInsets()`, `AsyncStorage` (feed order/view mode persistence, keys `@awefull_people:home_feed_order_mode` / `_view_mode`), dynamic `import('expo-location')` for last-known-position (used only for sunset-based greeting elsewhere; silently no-ops without permission).
- `entryLocationCache` utils (`loadEntryLocationCache`, `geocodeAndCacheEntries`) — offline-safe cached geocoding for the country/city text shown on feed cards.

## Interactions
- `FlashList` (Shopify) for the feed — `getItemType` gives trip Recap/Stories cards their own recycling pool from entry/journal cards, since they render structurally different trees (LinearGradient+BlurView vs. plain photo) and were observed to render blank when recycled into the wrong pool.
- Snap-scroll offsets (`feedSnapOffsets`) computed so cards land at 17% of viewport height — used for both pull-based UX polish and to auto-scroll to a card before showing its long-press overlay (`feedCardScrollOffsets`).
- All card animations (opacity, translateY, scale) use the legacy `Animated` API, one `{opacity, translateY, scale}` triple per item ID stored in a `Map` ref (`cardAnimations`), pruned when items are removed.
- Long-press (`delayLongPress={450}`) → measures the card's window position (`measureInWindow`) to position the action overlay exactly over it.
- Pull-to-refresh has a fully custom indicator (not the native `RefreshControl` spinner) — `refreshRevealAnim` fades a custom pill in/out.

## Theming notes
Fully theme-aware via `useTheme()` (`colors.background`, `colors.textPrimary/Secondary`, `colors.border`, `colors.surfaceVariant`, `colors.stone600`, `colors.white`). `isDark` additionally branches specific effects: `BlurView` `intensity`/`tint`, feed-card border width (`isDark ? 0 : hairlineWidth`, since dark cards rely on photo contrast rather than a border), and the Trip Header card's fixed light-gray (`#D9D9DE`) background in dark mode (an intentional "paper card" look rather than following the dark surface color).

## Reference UI skeleton
Full top-level structure (feed mode / gallery mode / pull-to-refresh / long-press overlay / floating capture pill), collapsed at the card level: `renderEntryCard`'s feed-mode branch is shown as the one representative card; `renderJournalCard` and `renderTripStoryCard` follow the same shape (measured layout ref for long-press, `Animated.View` wrapper with per-item `{opacity, translateY, scale}`, `TouchableOpacity` → `navigation.navigate`) but with journal-card/trip-story-specific inner content — see their own `file:line` for exact JSX. The two `ListHeaderComponent` cards (Travel Mode glass card vs. active-trip Trip Header card) are condensed since both share the same title/state-text/`Switch` row, differing only in background treatment:

```tsx
// source: src/screens/HomeScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const HomeScreen = () => {
  // entries/journals/tripAlbums from AppContext, feedViewMode ('feed'|'gallery'), feedRenderItems
  // (shuffled/ordered mix of entry/journal/tripStory items), longPress*, isTransitioningToNotes,
  // capturePillVisibility, cardAnimations (Map<id, {opacity, translateY, scale}>) — see :201-529

  const renderCard = (item, sessionSeed) => {
    if (item.type === 'entry') return renderEntryCard(item.data, sessionSeed, column, loadMedia, galleryGetPriority);
    if (item.type === 'journal') return renderJournalCard(item.data, sessionSeed);
    return renderTripStoryCard(item.data); // gradient+BlurView promo card — see :1444-1566
  };

  // One representative card renderer — feed-mode branch only (gallery-mode branch at :1316+ swaps
  // aspect ratio + long-press positioning but reuses the same TouchableOpacity/measure pattern):
  const renderEntryCard = (entry, sessionSeed, column, loadMedia, galleryGetPriority) => {
    let anim = cardAnimations.current.get(entry.id) ?? { opacity: new Animated.Value(1), translateY: new Animated.Value(0), scale: new Animated.Value(1) };
    const showLongPress = () => { /* measureInWindow → setLongPressCardLayout/ActiveEntryId, scale to 1.05, fade overlay in — :1165-1183 */ };

    if (isFeed) {
      const hasPhoto = entry.mediaType === 'photo' && entry.photoPaths?.length > 0;
      return (
        <Animated.View key={entry.id} style={[{ width: cardWidth, marginBottom: FEED_CARD_MARGIN_BOTTOM, borderRadius: 16, overflow: 'hidden', aspectRatio }, feedCardBorderStyle, { opacity: anim.opacity, transform: [{ translateY: anim.translateY }, { scale: anim.scale }] }]}>
          <View ref={(el) => entryCardLayoutRefs.current.set(entry.id, el)} collapsable={false} style={{ flex: 1 }}>
            <TouchableOpacity style={{ flex: 1 }} onPress={() => navigation.navigate('EntryDetail', { entryId: entry.id })} onLongPress={handleLongPress} delayLongPress={450}>
              {hasPhoto ? <EntryPhotoImage uri={photoUri} priority={imgPriority} /> : <FeedCardPoster mediaType={entry.mediaType} />}
              {/* gradient scrim + country/city/date/title overlay, GradientText instead of Text when entry.isBookmarked — see :1211-1316 */}
            </TouchableOpacity>
          </View>
        </Animated.View>
      );
    }
    // gallery-mode branch: same card, narrower aspect via getGalleryAspectRatio(entry), column-aware long-press anchor — :1316-1377
  };

  return (
    <View style={[styles.container, { backgroundColor: colors.background }]}>
      {feedViewMode === 'feed' ? (
        <>
          {shuffledItems.length === 0 && (
            <View style={StyleSheet.absoluteFillObject} pointerEvents="none">
              {!loadTimedOut ? (
                <><LoadingSpinner size="small" /><Text style={styles.loadingSubtext}>Preparing your memories...</Text></>
              ) : (
                <Text style={styles.placeholderText}>{tripActive && !showAllMemories ? 'Capture Travel Memories To See Them Here!' : 'Your Photos And Experiences Will Be Shown Here!'}</Text>
              )}
            </View>
          )}
          <Animated.View style={[styles.feed, { opacity: feedOpacity }]}>
            <FlashList
              ref={flashListRef}
              data={feedRenderItems}
              extraData={feedListExtraData}
              keyExtractor={(item) => feedItemKey(item)}
              getItemType={(item) => (item.type === 'tripStory' ? `tripStory-${item.data.kind}` : item.type)} // see Key technique(s)
              drawDistance={screenHeight * 2}
              onViewableItemsChanged={onFeedViewableItemsChanged.current}
              renderItem={({ item }) => renderCard(item, sessionSeed)}
              ListHeaderComponent={
                <View onLayout={(e) => setHeaderHeight(e.nativeEvent.layout.height)} collapsable={false}>
                  <View onLayout={(e) => setHeightAboveMemories(e.nativeEvent.layout.height)} collapsable={false}>
                    {!tripHeaderData ? (
                      // Travel Mode glass card — BlurView (iOS) / translucent View fallback (Android),
                      // title + "On"/"Off" state text + Switch — full JSX at :1628-1690
                      <View style={styles.travelModeGlassCard}>
                        {Platform.OS === 'ios' ? <BlurView intensity={isDark ? 48 : 62} tint={isDark ? 'dark' : 'light'} style={StyleSheet.absoluteFillObject} /> : <View style={[StyleSheet.absoluteFillObject, { backgroundColor: isDark ? 'rgba(32,30,30,0.94)' : 'rgba(248,248,248,0.97)' }]} />}
                        <View style={styles.travelModeGlassRow}>
                          <View style={styles.travelModeGlassTextBlock}>
                            <Text style={styles.travelModeGlassTitle}>Travel Mode</Text>
                            <Text style={styles.travelModeGlassState}>{travelModeEnabled ? 'On' : 'Off'}</Text>
                          </View>
                          <Switch value={travelModeEnabled} onValueChange={handleTravelModeToggle} trackColor={{ false: isDark ? 'rgba(255,255,255,0.24)' : '#D1D5DB', true: '#34C759' }} />
                        </View>
                      </View>
                    ) : (
                      // Trip Header card — replaces the glass card; fixed light-gray bg even in dark
                      // mode ("paper card" look), same title/Switch row + "Current Trip: {names}" + "Day X of Y" — full JSX at :1691-1749
                      <View style={[styles.tripHeaderCard, { backgroundColor: isDark ? '#D9D9DE' : 'rgba(248,248,248,0.97)' }]}>
                        <View style={styles.tripHeaderTopRow}>
                          <View style={styles.tripHeaderTopTextBlock}>
                            <Text style={styles.travelModeGlassTitle}>Travel Mode</Text>
                            <Text style={styles.tripHeaderState}>On</Text>
                          </View>
                          <Switch value={travelModeEnabled} onValueChange={handleTravelModeToggle} />
                        </View>
                        <Text style={styles.tripHeaderTitle} numberOfLines={2}>{tripHeaderData.title /* "Current Trip: {names}" */}</Text>
                        {/* "Day X of Y" / "Day X" below — :1750+ */}
                      </View>
                    )}
                    {showHomeMap && <InteractiveMap entries={entries} onEntryPress={handleEntryPress} collapsed={mapCollapsed} onToggleCollapse={() => setMapCollapsed(!mapCollapsed)} showEntryPins={false} />}
                  </View>
                  <View style={styles.memoriesHeaderRow}>
                    <View style={{ flex: 1 }}>
                      <Text style={styles.memoriesTitle}>{tripActive && !showAllMemories ? "This Trip's Memories" : 'Your Memories'}</Text>
                      {tripActive && <TouchableOpacity onPress={() => setShowAllMemories(v => !v)}><Text style={styles.locationLabel}>{showAllMemories ? "Show this trip's memories" : 'Show all memories'}</Text></TouchableOpacity>}
                    </View>
                    <HomeMemoriesFeedControls orderMode={feedOrderMode} viewMode={feedViewMode} onOrderChange={setFeedOrderMode} onViewChange={setFeedViewMode} />
                  </View>
                </View>
              }
            />
          </Animated.View>
        </>
      ) : (
        // Gallery mode: same header content inside a plain ScrollView, cards rendered by
        // HomeGalleryMasonryGrid (two-column masonry) instead of FlashList — see :1955-2008
        <ScrollView>
          {/* ...same ListHeaderComponent content inlined... */}
          <HomeGalleryMasonryGrid ref={galleryGridRef} items={feedRenderItems} galleryCardWidth={galleryCardWidth} getTileHeight={getGalleryTileHeight} sessionSeed={sessionSeed} emptyPlaceholder={galleryEmptyPlaceholder} renderCard={renderCard} />
        </ScrollView>
      )}

      {/* Custom pull-to-refresh indicator — not the native RefreshControl spinner */}
      <Animated.View pointerEvents="none" style={[styles.pullIndicatorWrapper, { top: pullIndicatorTop }]}>
        <Animated.View style={[StyleSheet.absoluteFill, { opacity: refreshRevealAnim, alignItems: 'center', justifyContent: 'center' }]}>
          <LoadingSpinner size="small" />
        </Animated.View>
      </Animated.View>

      {/* Long-press overlay: dimming layer with a "cutout" so the selected card stays bright,
          anchored to the card's measured window position, action row (delete/edit/journal) — see :2035-2260 */}
      {longPressActiveEntryId && longPressCardLayout && (
        <View style={StyleSheet.absoluteFillObject} pointerEvents="box-none">
          {/* dim + cutout mask sized to longPressCardLayout * 1.05 scale + buffer */}
          {/* action icons: create journal → NotesEditor, edit → EntryEdit, delete → confirm + deleteEntry */}
        </View>
      )}

      {/* Floating capture pill — bottom-right, glassmorphic; expands into a full-width sheet
          during the notes-editor transition instead of navigating immediately */}
      <Animated.View
        pointerEvents={isTransitioningToNotes ? 'auto' : hidden ? 'box-none' : 'none'}
        style={
          isTransitioningToNotes
            ? { position: 'absolute', bottom: 0, left: 0, right: 0, zIndex: 3000, backgroundColor: colors.background,
                transform: [{ translateY: notesTransitionProgress.interpolate({ inputRange: [0, 1], outputRange: [0, -screenHeight + homeCaptureTransitionStart] }) }],
                height: notesTransitionProgress.interpolate({ inputRange: [0, 1], outputRange: [homeCaptureTransitionStart, screenHeight] }) }
            : { position: 'absolute', left: 0, right: 0, bottom: homeCaptureBottomHidden, zIndex: HOME_CAPTURE_BAR_ZINDEX }
        }
      >
        {!isTransitioningToNotes && (
          <Animated.View onLayout={onHomeCapturePillLayout} style={{ width: homeCapturePillWidth, alignSelf: 'flex-end', opacity: capturePillOpacity, transform: [{ translateY: capturePillTranslateY }] }}>
            <View style={styles.capturePillOuter}>
              {Platform.OS === 'ios' ? <BlurView intensity={isDark ? 52 : 64} tint={isDark ? 'dark' : 'light'} style={StyleSheet.absoluteFillObject} /> : <View style={[StyleSheet.absoluteFillObject, { backgroundColor: isDark ? 'rgba(32,30,30,0.94)' : 'rgba(248,248,248,0.97)' }]} />}
              <View style={styles.capturePillRow}>
                <View style={styles.captureActionsRow}>
                  <TouchableOpacity style={styles.capturePenSlot} onPress={handleNotePlaceholderPress}>
                    <PenIcon width={20} height={20} color={colors.textPrimary} />
                  </TouchableOpacity>
                  <Animated.View style={{ transform: [{ scale: cameraButtonScale }] }}>
                    <TouchableOpacity style={styles.captureCameraSlot} onPress={handleCameraPress} onPressIn={handleCameraPressIn} onPressOut={handleCameraPressOut}>
                      <CameraIcon width={24} height={23} color={colors.textPrimary} />
                    </TouchableOpacity>
                  </Animated.View>
                </View>
              </View>
            </View>
          </Animated.View>
        )}
      </Animated.View>
    </View>
  );
};
```

**Key technique(s):** the FlashList recycling-pool fix (`:1607-1620` — `getItemType`) is worth knowing before adding any new heterogeneous card type to a virtualized list elsewhere in the app: Recap/Stories cards render structurally different trees (`LinearGradient`+`BlurView` vs. a plain bordered photo) despite sharing `item.type === 'tripStory'`, so they're given a distinct recycling pool per `kind` — without this, FlashList reused entry-card cells for Stories cards and they rendered blank/invisible. The notes-editor transition reuses the same `Animated.Value` (`notesTransitionProgress`) to drive both a `translateY` and a `height` interpolation simultaneously, turning the compact pill into a full-width sheet without swapping components.

## Screenshot
_Not yet captured for this pass (6 screenshots already exist under `screens/` from an earlier extraction — see `home-feed-list.png`, `home-grid-pinterest.png`, `home-grid-dark.png` — but none show the Travel Mode glass card, the active Trip Header card, the loading state, or the long-press overlay). TODO: capture light + dark for: empty state, Travel-Mode-off glass card, active-trip Trip Header card, long-press overlay open, notes-editor transition mid-animation._

## Open questions / follow-ups
- The file mixes the legacy `Animated` API throughout (not Reanimated), unlike some other screens/components in the app (e.g. `AppNavigator`'s `ActiveIndicator` also uses legacy `Animated`, but other newer components may use Reanimated) — worth flagging to a designer/engineer if a future refactor wants one consistent animation library.
- `HOME_CAPTURE_USE_WIDE_PILL` is a hardcoded `false` with a full alternate "wide pill" JSX branch kept dead in the tree for "future reuse" (per its own comment) — a screenshot can't show this variant since it's currently unreachable in the running app.
