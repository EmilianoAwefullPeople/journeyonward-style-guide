# Entry Detail

**Route name:** `EntryDetail`
**File:** `src/screens/EntryDetailScreen.tsx` (comment cites Figma node `342-81`)
**Type:** Stack screen (`presentation: 'card'`)

## Purpose
The full-screen read view for a single captured entry (experience or insight): media, title, location/date, and — progressively disclosed — description, emotion, weather/temperature/season/altitude, linked album, and branch journals. The "go deeper on one memory" screen for the Retrospective and Reflective personas.

## Entry points & navigation
- Entry: from `HomeScreen`'s `handleEntryPress` (`navigation.navigate('EntryDetail', { entryId })`), from Review/gallery cards, from trip album detail, etc. — the generic "open this entry" destination app-wide.
- `route.params`: `{ entryId: string }`.
- Exit / further navigation:
  - Back arrow → `navigation.goBack()`.
  - 3-dot menu → **Journal** (`navigate('NotesEditor', { entryId })` — creates a "branch journal" linked to this entry), **Edit** (`navigate('EntryEdit', { entryId })`), **Download image to Photos** (photo entries only, via `expo-media-library`), **Delete** (confirm `Alert`, then `deleteEntry(entryId)` + `goBack()`).
  - "Write About This Experience" button (bottom) and a drag-up gesture on the same button both → `navigate('NotesEditor', { entryId })`.
  - Tapping the linked album's stat cell → `navigate('TripAlbumDetail', { albumId })`.
  - Tapping the branch-journals stat cell → `navigate('JournalDetail', { journalId })` directly if exactly one, or opens an in-screen modal picker if multiple.
  - Tapping the photo → opens a full-screen `Modal` photo viewer (paged, zoomable, with a `X / N` counter).

## Layout anatomy
1. Fixed (non-scrolling) back arrow + "..." menu button, top corners, hidden while the full-screen photo viewer is open.
2. Scrollable content:
   - Media area — photo carousel (paged `FlatList` + pagination dots) if `photoPaths.length > 1`, single `ZoomablePhoto` if exactly one, or an emoji placeholder (🎥/🎵/📷) for video/audio/no-media entries.
   - Title — plain text, or (if `entry.isBookmarked`) a gradient-filled title using `MaskedView` + `LinearGradient`, with a small compass icon prefix.
   - Location line (pin icon) + date line (calendar icon).
   - "More Info..." / "Show less" toggle link — only rendered if there's anything to show (description, emotion, weather/temp/season/altitude, linked album, branch journals, or emotion note).
   - Expanded-only: quoted italic description, formatted emotion sentence (see below), a 6-cell data grid (Temperature / Weather / Season / Altitude / Album / Branch Journals — each a "—" placeholder when absent), and the raw emotion note text.
   - "Write About This Experience" pill button, centered, bottom of scroll content.
3. Two `Modal`s outside the scroll flow: full-screen photo viewer (opaque black backdrop) and a branch-journal picker (translucent backdrop, centered card).
4. 3-dot dropdown menu — absolutely positioned, closes on backdrop tap or option select.

## States
- **Entry not found** (`!entry`, e.g. deleted from another screen) → back arrow + "Entry not found" / "This entry may have been deleted" text, nothing else.
- **Media type branch**: photo (carousel or single), video placeholder, audio placeholder, or generic "No Media" placeholder — determined by `entry.mediaType` and whether `photoPaths` is non-empty.
- **`showMoreInfo`**: collapsed (just the "More Info..." link) vs. expanded (full data grid + description + emotion + emotion note) — computed from `hasMoreInfo`, an OR of 9 different optional fields, so the link itself is hidden entirely when there's truly nothing extra to show.
- **Bookmarked vs. not**: title renders via gradient-masked text (bookmarked) vs. plain `Text` (not).
- **Carousel interaction** (`carouselInteracting`): suppresses tap-to-open-viewer for ~160ms after a drag/momentum scroll ends, so a swipe-to-next-photo doesn't accidentally register as a tap-to-zoom.
- **Inline zoom active** (`inlinePhotoZoomed`) or **full-screen photo viewer open** (`isPhotoViewerVisible`): disables the outer `ScrollView`'s `scrollEnabled` so pinch-zoom gestures on the photo don't fight page scroll.
- **Menu open/closing**: `menuVisible` / `menuExiting` two-state pattern — closing plays a 150ms fade+scale-down before actually unmounting, so `closeMenuAnimated(callback)` can chain a navigation action after the close animation finishes rather than cutting it short.
- **Downloading image** (`isDownloadingImage`): download menu option label switches to "Saving…" and disables itself; shows a permission-needed alert if photo-library access isn't granted.
- **Branch journals**: 0 → grid cell shows "0" (no tap action); 1 → tapping the grid cell navigates directly to that journal; 2+ → tapping opens the in-modal picker list.

## Data & context dependencies
- `useAppContext()` → `entries`, `tripAlbums`, `journals`, `deleteEntry`.
- `useTheme()` → `colors`, `isDark` (the data-grid value/label colors and the "Write About" button background/border both branch explicitly on `isDark` rather than relying purely on theme tokens).
- `formatLocation`/`formatEntryDate`/`formatTemperature` (`src/utils/dateUtils.ts`) — location formatting is async (likely does reverse-geocoding), loaded in an effect and animated in via `metadataOpacity`.
- `getEmojiForEmotionLabel` (`src/utils/emotionUtils.ts`) for the emotion sentence's leading emoji.
- `expo-media-library` for the "Download image to Photos" action.

## Interactions
- `PanResponder` (not Gesture Handler) on the "Write About This Experience" button: a drag-up past 60px, or a near-stationary tap, both trigger the same NotesEditor navigation — a deliberately loose gesture so either a tap or an upward swipe on that button works.
- Mount-in animation: `screenOpacity` (300ms), `contentTranslateY` (20→0, 350ms, 50ms delay), `imageScale` (0.95→1, 400ms, 100ms delay) — all `Animated.parallel`, staggered via delay rather than sequence.
- Photo carousel and full-screen viewer both use `ZoomablePhoto` (`src/components/common/ZoomablePhoto.tsx`) — pinch/double-tap zoom is delegated to that shared component; this screen only tracks whether it's currently zoomed (to gate outer scroll) via `onZoomStateChange`.
- 3-dot menu: `menuOverlayOpacity` fade + `menuDropdownScale` (0.92→1) scale-in on open (200/220ms), reversed on close (150/150ms) with an `Easing.in`/`Easing.out` split — the classic "faster close, springier open" feel used elsewhere in the app (cf. `AppNavigator`'s plus-menu).

## Theming notes
Uses `useTheme()` broadly, but several elements have explicit `isDark` branches rather than pure token usage: data-grid text color (`isDark ? '#e4e4e4' : colors.textPrimary`), the "Write About" button's background (`isDark ? COLORS.buttonGray : colors.surfaceVariant`), and its border color — all suggesting these were hand-tuned per theme rather than left to fall out of the token system automatically.

## Reference UI skeleton
Full render tree, collapsed where a section is a repeated list (the 6-cell data grid shows 2 representative cells; the 4 near-identical menu options show as one pattern):

```tsx
// source: src/screens/EntryDetailScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const EntryDetailScreen = () => {
  // entry lookup, showMoreInfo, menuVisible/menuExiting, isPhotoViewerVisible, inlinePhotoZoomed,
  // carouselInteracting, isDownloadingImage, branchJournalListVisible, mount-in animation values — see :29-580

  if (!entry) {
    return (
      <View style={styles.container}>
        <TouchableOpacity style={[styles.backButton, { top: (107.07 - 74 + 20) * scaleFactor, left: 25 * scaleFactor }]} onPress={handleBack}>
          <BackArrow color={colors.textPrimary} />
        </TouchableOpacity>
        <View style={styles.errorContainer}>
          <Text style={styles.errorText}>Entry not found</Text>
          <Text style={styles.errorSubtext}>This entry may have been deleted</Text>
        </View>
      </View>
    );
  }

  return (
    <Animated.View style={[styles.container, { opacity: screenOpacity }]}>
      {/* 3-dot dropdown — absolute, above everything, closes on backdrop tap */}
      {(menuVisible || menuExiting) && (
        <Animated.View style={[styles.menuOverlay, { opacity: menuOverlayOpacity }]} pointerEvents="box-none">
          <TouchableWithoutFeedback onPress={() => closeMenuAnimated()}><View style={StyleSheet.absoluteFill} /></TouchableWithoutFeedback>
          <Animated.View style={[styles.menuDropdown, { top: menuDropdownTop, right: 25 * scaleFactor, transform: [{ scale: menuDropdownScale }] }]}>
            {/* 4 options, each an icon + label TouchableOpacity: Journal, Edit, Download (photo entries only,
                label flips to "Saving…" while isDownloadingImage), Delete (red icon/text) — see :626-670 */}
          </Animated.View>
        </Animated.View>
      )}

      {/* Fixed back + "..." — hidden while the full-screen photo viewer is open */}
      {!isPhotoViewerVisible && (
        <>
          <TouchableOpacity style={[styles.backButton, { top: insets.top + 20, left: 25 * scaleFactor }]} onPress={handleBack}>
            <BackArrow color={colors.textPrimary} />
          </TouchableOpacity>
          <TouchableOpacity style={[styles.menuButton, { top: insets.top + 20, right: 25 * scaleFactor }]} onPress={toggleMenu}>
            <Text style={styles.menuButtonText}>...</Text>
          </TouchableOpacity>
        </>
      )}

      <ScrollView style={styles.scrollView} scrollEnabled={!inlinePhotoZoomed && !isPhotoViewerVisible}>
        <Animated.View style={{ transform: [{ translateY: contentTranslateY }] }}>
          <View style={[styles.imageWrapperWithHeader, { minHeight: imageHeight }]}>
            <Animated.View style={[styles.imageContainer, { width: imageWidth, height: imageHeight, transform: [{ scale: imageScale }] }]}>
              {imageUri ? (
                entry.photoPaths.length > 1 ? (
                  <View style={{ flex: 1 }}>
                    <FlatList
                      data={entry.photoPaths}
                      horizontal
                      pagingEnabled
                      scrollEnabled={!inlinePhotoZoomed}
                      renderItem={({ item, index }) => (
                        <ZoomablePhoto uri={item} isActive={index === activeSlide} suppressTap={carouselInteracting} onTap={() => openPhotoViewer(index)} onZoomStateChange={/* :737-741 */ onZoom} />
                      )}
                      // carousel drag/momentum handlers suppress tap-to-zoom for ~160ms after a swipe — see :748-774
                    />
                    <View style={styles.paginationContainer}>
                      {entry.photoPaths.map((_, index) => (
                        <View key={index} style={[styles.paginationDot, index === activeSlide ? styles.paginationDotActive : styles.paginationDotInactive]} />
                      ))}
                    </View>
                  </View>
                ) : (
                  <ZoomablePhoto uri={imageUri} width={imageWidth} height={imageHeight} onTap={() => openPhotoViewer(0)} onZoomStateChange={setInlinePhotoZoomed} />
                )
              ) : (
                <View style={styles.mediaPlaceholder}>
                  <Text style={styles.mediaIcon}>{entry.mediaType === 'video' ? '🎥' : entry.mediaType === 'audio' ? '🎵' : '📷'}</Text>
                  <Text style={styles.mediaLabel}>{entry.mediaType === 'video' ? 'Video' : entry.mediaType === 'audio' ? 'Audio' : 'No Media'}</Text>
                </View>
              )}
            </Animated.View>
          </View>

          <View style={styles.sectionPadding}>
            {entry.isBookmarked ? (
              <MaskedView
                style={{ flexDirection: 'row', alignItems: 'center', flex: 1 }}
                maskElement={
                  <View style={styles.titleRow}>
                    <Ionicons name="compass-outline" size={15 * scaleFactor} color="white" />
                    <Text style={[styles.title, { color: 'white' }]} numberOfLines={2}>{entry.title || '(Untitled)'}</Text>
                  </View>
                }
              >
                <LinearGradient colors={[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]} start={{ x: 0, y: 0 }} end={{ x: 1, y: 0 }} style={StyleSheet.absoluteFill} />
              </MaskedView>
            ) : (
              entry.title && <Text style={styles.title} numberOfLines={3}>{entry.title}</Text>
            )}

            {locationLabel && <View style={styles.metaRow}><LocationIcon color={colors.textPrimary} /><Text style={styles.locationDate}>{locationLabel}</Text></View>}
            {dateLabel && <View style={styles.metaRow}><CalendarIcon color={colors.textPrimary} /><Text style={styles.locationDate}>{dateLabel}</Text></View>}

            {/* hasMoreInfo = OR of 9 optional fields; link hidden entirely if none are present — see :882-931 */}
            {hasMoreInfo && (
              showMoreInfo ? (
                <>
                  <TouchableOpacity onPress={() => setShowMoreInfo(false)}><Text style={styles.showMoreLink}>Show less</Text></TouchableOpacity>
                  {entry.description && <Text style={styles.description}>"{entry.description.trim()}"</Text>}
                  {emotionText && <Text style={styles.emotion}>{emotionText}</Text>}
                </>
              ) : (
                <TouchableOpacity onPress={() => setShowMoreInfo(true)}><Text style={styles.showMoreLink}>More Info...</Text></TouchableOpacity>
              )
            )}
          </View>

          {/* 6-cell data grid, only when showMoreInfo — 2 representative cells shown, other 4
              (Weather, Season, Altitude, Branch Journals) follow the same value/label pattern — see :934-1009 */}
          {showMoreInfo && (
            <>
              <View style={[styles.sectionPadding, styles.dataGridFull]}>
                <View style={styles.dataGridCell}>
                  <Text style={styles.dataGridValue}>{entry.temperatureCelsius != null ? formatTemperature(entry.temperatureCelsius) : '—'}</Text>
                  <Text style={styles.dataGridLabel}>Temperature</Text>
                </View>
                <View style={styles.dataGridCell}>
                  {linkedAlbum ? (
                    <TouchableOpacity onPress={handleAlbumPress}>
                      <Text style={styles.dataGridValue}>Saved to 1</Text>
                      <Text style={styles.dataGridLabel}>Album</Text>
                    </TouchableOpacity>
                  ) : (
                    <><Text style={styles.dataGridValue}>—</Text><Text style={styles.dataGridLabel}>Album</Text></>
                  )}
                </View>
                {/* ...Weather, Season, Altitude, Branch Journals cells */}
              </View>
              {entry.emotionNote && <View style={styles.sectionPadding}><Text style={styles.description}>{entry.emotionNote.trim()}</Text></View>}
            </>
          )}

          <View style={[styles.sectionPadding, { alignItems: 'center' }]}>
            <TouchableOpacity
              style={styles.writeAboutButton}
              onPress={() => navigation.navigate('NotesEditor', { entryId })}
              {...journalHandleResponder.panHandlers}
            >
              <Ionicons name="create-outline" size={16} color={colors.textPrimary} />
              <Text style={styles.writeAboutButtonText}>Write About This Experience</Text>
            </TouchableOpacity>
          </View>
        </Animated.View>
      </ScrollView>

      {/* Full-screen photo viewer — paged FlatList, opaque black backdrop, X/N counter */}
      <Modal visible={isPhotoViewerVisible} animationType="fade" onRequestClose={closePhotoViewer}>
        <View style={styles.photoViewerBackdrop}>
          <View style={styles.photoViewerHeader}>
            <Text style={styles.photoViewerCounter}>{`${viewerIndex + 1} / ${entry.photoPaths.length}`}</Text>
            <TouchableOpacity onPress={closePhotoViewer}><Text style={styles.photoViewerCloseText}>Close</Text></TouchableOpacity>
          </View>
          <FlatList
            data={entry.photoPaths}
            horizontal
            pagingEnabled
            scrollEnabled={!fullscreenPhotoZoomed}
            renderItem={({ item, index }) => (
              <View style={[styles.fullScreenPhotoSlide, { width: screenWidth }]}>
                <ZoomablePhoto uri={item} resizeMode="contain" fallbackBackgroundColor="#000000" isActive={index === viewerIndex} onZoomStateChange={/* :1073-1077 */ onZoom} />
              </View>
            )}
          />
        </View>
      </Modal>

      {/* Branch-journal picker — translucent backdrop, centered card, opens only when 2+ branch journals */}
      <Modal visible={branchJournalListVisible} transparent animationType="fade" onRequestClose={() => setBranchJournalListVisible(false)}>
        <TouchableWithoutFeedback onPress={() => setBranchJournalListVisible(false)}>
          <View style={styles.modalBackdrop}>
            <TouchableWithoutFeedback>
              <View style={styles.modalCard}>
                <Text style={styles.modalTitle}>Branch Journals</Text>
                {branchJournals.length > 0 ? (
                  <ScrollView>
                    {branchJournals.map((journal) => (
                      <TouchableOpacity key={journal.id} onPress={() => navigation.navigate('JournalDetail', { journalId: journal.id })}>
                        <Text numberOfLines={1}>{journal.title || 'Journal'}</Text>
                        {journal.content && <Text numberOfLines={2}>{journal.content}</Text>}
                      </TouchableOpacity>
                    ))}
                  </ScrollView>
                ) : (
                  <Text>No branch journals found for this entry.</Text>
                )}
              </View>
            </TouchableWithoutFeedback>
          </View>
        </TouchableWithoutFeedback>
      </Modal>
    </Animated.View>
  );
};
```

**Key technique(s):** the gradient-masked bookmarked title (`:827-850`) — a `MaskedView` whose mask is the title text/icon and whose visible content is a `LinearGradient` — is reusable anywhere a "special" title needs a brand gradient fill instead of a solid color, since RN can't apply a gradient to `Text` color directly. `hasMoreInfo` (`:883-893`) is an OR of 9 independent optional fields, so the "More Info..." link itself disappears entirely (not just collapses empty) when an entry truly has nothing extra to show.

## Screenshot
_Not yet captured. TODO: capture light + dark for: single photo, multi-photo carousel with pagination dots, video/audio placeholder, collapsed vs. expanded "More Info" data grid, bookmarked gradient title, 3-dot menu open, full-screen photo viewer, branch-journals picker modal, entry-not-found error state._

## Open questions / follow-ups
- "Saved to 1" is a hardcoded string for the Album stat cell whenever `linkedAlbum` exists (it doesn't count anything — an entry only ever has one `albumId`) — worth confirming with product/design whether that label is intentional phrasing or a placeholder that should instead show the album's name.
- The error-state back button (`!entry` branch) computes its `top` offset with a comment referencing "Image starts at 107.07, container at 74" — a hand-tuned magic number carried over from a Figma layout; a screenshot of this specific error state would help confirm it still lines up correctly on current device sizes.
