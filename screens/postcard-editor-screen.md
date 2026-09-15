# Postcard Editor Screen

**Route name:** `PostcardEditor`
**File:** `src/screens/PostcardEditorScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
The actual postcard-composition canvas: fills a chosen template's photo slot(s), lets the user pick/reframe photos, edit text labels, and adjust template-specific options (date range, data category, location-tag mode), then save the postcard and/or export it to the camera roll. Serves the Travel Sharer persona.

## Entry points & navigation
- From `PostcardTemplatesScreen` → `navigation.navigate('PostcardEditor', { templateId })` (new postcard).
- From `SavedPostcardsScreen` → `navigation.navigate('PostcardEditor', { postcardId })` (edit existing; template is re-derived from the saved postcard, not passed).
- Params: `{ templateId?: string; postcardId?: string }`.
- Exits via back chevron (`navigation.goBack()`); no forward navigation — this is a leaf/terminal screen in the flow.

## Layout anatomy
1. Header: back chevron, flexible spacer, then two icon actions (bookmark = save, download = export to camera roll).
2. Aspect-ratio row: three text buttons (`1:1`, `4:5`, `9:16`).
3. Canvas area: centered `templateWrapper` (drop-shadowed) rendering either the legacy T-11 diptych component or the generic multi-slot `TemplateComponent`, both loaded dynamically.
4. Template-specific control row, conditionally rendered below the canvas:
   - Month-calendar templates (T-07): a tappable "change date range" row.
   - Triptych-stamps template (T-06): a row of icon buttons to pick the displayed data category (e.g. temperature vs. something else).
   - Location-tag template (T-04): a row of tag-mode buttons, and (if mode is "location") a second row of granularity buttons.
5. Overlays (conditionally mounted on top of everything): `PhotoPickerSheet` (slot photo picker), `TextEditOverlay` (full-screen text edit for a label), `DateRangeSheet` (two stacked `DateTimePicker`s for the month-calendar range).

## States
- **Template loading**: `TemplateComponent === null` (dynamic `require` not yet resolved) → centered `ActivityIndicator`.
- **Template load error**: `templateLoadError !== null` (component didn't export a renderable type) → error text + "use back button" hint.
- **Loaded, editing**: normal canvas with all template controls.
- **Editing a slot's photo**: `activeSlot !== null` → `PhotoPickerSheet` visible.
- **Editing a text label**: `activeTextSlot !== null` → full-screen `TextEditOverlay`.
- **Editing the date range** (month-calendar template only): `rangeSheetVisible` → `DateRangeSheet`.
- **Saving**: `isSaving` → bookmark icon replaced by `ActivityIndicator`; save button disabled.
- **Already saved / no changes**: `isSaved` (current state's JSON snapshot matches `savedSnapshot`) → filled bookmark icon, save button disabled.
- **Downloading/exporting**: `isDownloading` → download icon replaced by spinner; `editingChromeSuppressed` set true so editing affordances (slot borders, etc.) hide from the exported capture.
- **Download disabled**: `canDownload` is false when slot 0 has no photo yet, or a save/download is already in flight.
- **Loading an existing postcard** (`postcardId` present): a mount-time effect finds it in `postcards`, corrects `rawTemplateId`, and rehydrates photos/transforms/labels/ratio/template data before the user sees anything editable.

## Data & context dependencies
- `useAppContext()` → `postcards`, `entries`, `createPostcard`, `updatePostcard`.
- `useTheme()` → `colors`.
- `useSafeAreaInsets()`, `useWindowDimensions()`.
- Template system helpers from `src/components/postcards/templates/templateLoader`: `normalizeTemplateId`, `getMaxSlotsForTemplate`, `loadTemplateComponent` (dynamic per-template `require`), `buildSlotsFromPostcard`, `packPostcardPhotoFields`.
- Template-specific utility modules: `monthCalendarUtils`, `triptychStampsUtils`, `locationTagUtils`, `diptychCaptionUtils`.
- `Analytics.onPostcardExported(tripId)` fire-and-forget call on successful export.
- Heavy native deps (`expo-media-library`, `react-native-view-shot`) are `require()`'d lazily inside `handleDownload`, not at module scope, to keep them out of `AppNavigator`'s eager init path.

## Interactions
- Ratio buttons: tap to switch `activeRatio`, recomputing the canvas's aspect ratio live.
- Slot tap → opens `PhotoPickerSheet` (photo reframing itself, e.g. pinch/pan, is inside the individual template components, not this screen — this screen only owns `transforms` state and passes `onTransformChange`/`onResetSlotFraming` callbacks down).
- Label tap → `TextEditOverlay`, a full-screen dark backdrop with an auto-focused, italic, yellow (`#FFE600`) single-line `TextInput`, 60-char max, dismissed via a "Done" button or the return key.
- Download: captures the template view via `react-native-view-shot`'s `captureRef` at a fixed 1080px-wide output, requests media-library permission first, then saves and persists the postcard with `downloaded: true`.
- Bookmark/save: persists via `createPostcard`/`updatePostcard` without exporting.

## Theming notes
- Screen chrome (header, ratio labels, control-row buttons) uses `useTheme()` colors throughout.
- `TextEditOverlay` and `DateRangeSheet` are deliberately NOT theme-aware — both render as fixed near-black full-screen backdrops with white/yellow text regardless of app theme, matching a "full-screen editing mode" convention distinct from normal screen chrome.
- The postcard canvas itself renders through the selected template component, whose visual style is fixed by design (independent of light/dark).

## Reference UI skeleton
Whole render tree, business logic stripped. `RATIO_OPTIONS = [{'1:1',1}, {'4:5',4/5}, {'9:16',9/16}]`. `templateWidth = screenWidth - SPACING.lg*2`.

```tsx
// source: src/screens/PostcardEditorScreen.tsx (skeleton — business logic stripped)

// ── TextEditOverlay sub-component (lines 92-131) ────────────────────────────
// Full-screen fixed-dark overlay for editing one text label. NOT theme-aware.
const TextEditOverlay = ({ value, onChange, onDone }) => (
  <View style={overlayStyles.backdrop}> {/* absoluteFill, bg: 'rgba(0,0,0,0.82)', zIndex:100 */}
    <View style={[overlayStyles.topBar, { paddingTop: insets.top + 8 }]}>
      <TouchableOpacity onPress={onDone}><AppText style={overlayStyles.doneText}>Done</AppText></TouchableOpacity>
    </View>
    <KeyboardAvoidingView style={overlayStyles.keyboardArea} behavior={Platform.OS === 'ios' ? 'padding' : 'height'}>
      <View style={overlayStyles.inputWrapper}> {/* flex:1, center */}
        <TextInput
          autoFocus value={value} onChangeText={onChange} onSubmitEditing={onDone}
          maxLength={60} selectionColor="#FFE600"
          style={overlayStyles.input} // color:'#FFE600', fontSize:24, italic, textAlign:'center'
        />
      </View>
    </KeyboardAvoidingView>
  </View>
);

// ── DateRangeSheet sub-component (lines 143-208, T-07 month-calendar only) ──
const DateRangeSheet = ({ visible, start, end, onChangeStart, onChangeEnd, onDone, onCancel }) => {
  if (!visible) return null;
  return (
    <View style={rangeSheetStyles.backdrop}> {/* absoluteFill, bg:'rgba(0,0,0,0.92)', zIndex:100 */}
      <View style={rangeSheetStyles.topBar}> {/* Cancel / "Select dates" / Done, row justify-between */}
        <TouchableOpacity onPress={onCancel}><AppText style={rangeSheetStyles.cancelText}>Cancel</AppText></TouchableOpacity>
        <AppText style={rangeSheetStyles.title}>Select dates</AppText>
        <TouchableOpacity onPress={onDone}><AppText style={rangeSheetStyles.doneText}>Done</AppText></TouchableOpacity>
      </View>
      <View style={rangeSheetStyles.body}> {/* flex:1, center, paddingTop:24 */}
        <AppText style={rangeSheetStyles.label}>Start date</AppText>
        <DateTimePicker value={start} mode="date" display="spinner" onChange={(_, d) => d && onChangeStart(d)} textColor="#fff" />
        <AppText style={[rangeSheetStyles.label, { marginTop: 24 }]}>End date</AppText>
        <DateTimePicker value={end} mode="date" minimumDate={minEnd} maximumDate={maxEnd} display="spinner" onChange={(_, d) => d && onChangeEnd(d)} textColor="#fff" />
        <AppText style={rangeSheetStyles.hint}>Trips can be {RANGE_MIN_DAYS}–{RANGE_MAX_DAYS} days long.</AppText>
      </View>
    </View>
  );
};

// ── Screen ───────────────────────────────────────────────────────────────────
// T-11 (Diptych) uses old-style topPhoto/bottomPhoto props (renderT11Template);
// every other template uses the uniform `slots[]` contract (renderNewTemplate).
return (
  <View style={styles.outer}> {/* flex:1, bg: colors.background */}
    <View style={styles.header}> {/* row, paddingTop: insets.top+8 */}
      <TouchableOpacity style={styles.backButton} onPress={() => navigation.goBack()}>
        <Ionicons name="chevron-back" size={24} color={colors.textPrimary} />
      </TouchableOpacity>
      <View style={styles.headerSpacer} /> {/* flex:1 */}
      <View style={styles.headerActions}> {/* row, gap:4 */}
        <TouchableOpacity style={styles.iconButton} onPress={() => handleSave(false)} disabled={isSaving || isSaved}>
          {isSaving ? <ActivityIndicator size="small" /> : <Ionicons name={isSaved ? 'bookmark' : 'bookmark-outline'} size={22} />}
        </TouchableOpacity>
        <TouchableOpacity style={styles.iconButton} onPress={handleDownload} disabled={!canDownload}>
          {isDownloading ? <ActivityIndicator size="small" /> : <Ionicons name="download-outline" size={22} />}
        </TouchableOpacity>
      </View>
    </View>

    <View style={styles.ratioRow}> {/* row, center, gap:24, paddingBottom:16 */}
      {RATIO_OPTIONS.map(({ label }) => (
        <TouchableOpacity key={label} onPress={() => setActiveRatio(label)}>
          <AppText style={[styles.ratioLabel, label === activeRatio && styles.ratioLabelActive]}>{label}</AppText>
        </TouchableOpacity>
      ))}
    </View>

    <View style={styles.canvasArea}> {/* flex:1, center, paddingHorizontal: SPACING.lg */}
      <View style={styles.templateWrapper} ref={templateRef}> {/* width: templateWidth, drop-shadow */}
        {templateLoadError ? (
          <AppText style={styles.templateErrorText}>{templateLoadError}</AppText>
        ) : TemplateComponent === null ? (
          <ActivityIndicator size="large" color={colors.textSecondary} /> // template still loading, see "Key technique"
        ) : isT11 ? (
          <TemplateComponent /* old prop contract */
            topPhoto={photos[0]} bottomPhoto={photos[1]}
            topLabel={labels[0]} bottomLabel={labels[1]}
            topTransform={transforms[0]} bottomTransform={transforms[1]}
            onSlotPress={...} onLabelPress={handleLabelPress}
            aspectRatio={aspectRatio} editingChromeSuppressed={editingChromeSuppressed}
          />
        ) : (
          <TemplateComponent /* uniform slots[] contract */
            slots={slots} onSlotPress={handleSlotPress}
            onTransformChange={handleTransformChange} onResetSlotFraming={handleResetSlotFraming}
            templateData={templateData} onTemplateDataChange={setTemplateData}
            aspectRatio={aspectRatio} editingChromeSuppressed={editingChromeSuppressed} allEntries={entries}
          />
        )}
      </View>
    </View>

    {/* Template-specific control rows — at most one of these three renders */}
    {isMonthCalendar && (
      <TouchableOpacity style={styles.monthSwitcherRow} onPress={handleOpenRangeSheet}>
        <Ionicons name="calendar-outline" size={18} />
        <AppText style={styles.monthSwitcherLabel}>{rangeLabel}</AppText>
        <Ionicons name="chevron-down" size={16} />
      </TouchableOpacity>
    )}
    {isTriptychStamps && (
      <View style={styles.dataCategoryRow}> {/* row, center, gap:10 */}
        {TRIPTYCH_DATA_CATEGORIES.map(({ key, icon }) => (
          <TouchableOpacity key={key} style={[styles.dataCategoryButton, isActive && styles.dataCategoryButtonActive]} onPress={() => handleChangeDataCategory(key)}>
            {/* 44x44 circle */} <Ionicons name={icon} size={22} />
          </TouchableOpacity>
        ))}
      </View>
    )}
    {isLocationTag && (
      <>
        <View style={styles.tagOptionsRow}> {/* LOCATION_TAG_MODES buttons */}
          {LOCATION_TAG_MODES.map(({ key, label, icon }) => (
            <TouchableOpacity key={key} style={[styles.tagOptionButton, isActive && styles.tagOptionButtonActive]} onPress={() => handleChangeTagMode(key)}>
              <Ionicons name={icon} size={16} /> <AppText style={styles.tagOptionText}>{label}</AppText>
            </TouchableOpacity>
          ))}
        </View>
        {selectedTagMode === 'location' && (
          <View style={[styles.tagOptionsRow, styles.tagOptionsRowLast]}> {/* LOCATION_GRANULARITIES buttons */}
            {LOCATION_GRANULARITIES.map(({ key, label }) => (
              <TouchableOpacity key={key} onPress={() => handleChangeTagGranularity(key)}><AppText>{label}</AppText></TouchableOpacity>
            ))}
          </View>
        )}
      </>
    )}

    <PhotoPickerSheet visible={activeSlot !== null} onClose={() => setActiveSlot(null)} onSelect={handlePhotoSelect} />
    {activeTextSlot !== null && <TextEditOverlay value={editingText} onChange={setEditingText} onDone={handleEditDone} />}
    <DateRangeSheet visible={rangeSheetVisible} start={draftRangeStart} end={draftRangeEnd} onChangeStart={handleDraftStartChange} onChangeEnd={setDraftRangeEnd} onDone={handleRangeDone} onCancel={handleRangeCancel} />
  </View>
);
```

**Key technique(s):**
- Dynamic per-template component loading is deferred to a post-mount `useEffect` (not required at module scope), so `AppNavigator`'s eager `require()` of this screen file never pulls in Reanimated (used by some template components) at app-boot time — `file:371-381`.
- Heavy native deps (`expo-media-library`, `react-native-view-shot`) are `require()`'d lazily inside `handleDownload` itself, for the same reason — `file:627-630`.
- `TextEditOverlay` and `DateRangeSheet` are hardcoded near-black/dark regardless of app theme — a deliberate "full-screen editing mode" convention distinct from the screen's normal theme-aware chrome.

## Screenshot
_Not yet captured. TODO: capture light + dark for: template loading spinner, a styled template mid-edit (e.g. T-01 or T-04) with a slot selected, the text-edit overlay, the date-range sheet (T-07), the triptych data-category row (T-06), the location-tag mode rows (T-04), and the saved/downloaded button states._

## Open questions / follow-ups
- `isSaved` is computed by deep-JSON-comparing the current editable state against the last-saved snapshot (`stateSnapshot`) rather than a dirty flag — worth confirming this comparison stays cheap as `templateData` grows more template-specific fields.
- T-11 (Diptych) uses an entirely separate prop contract (`topPhoto`/`bottomPhoto`/`topTransform`/…) from every other template's uniform `slots` array — flagged in-code as legacy ("T-11 (Diptych) uses the old-style props"); a future template addition should confirm whether new templates should migrate T-11 to the slots contract instead of perpetuating two shapes.
