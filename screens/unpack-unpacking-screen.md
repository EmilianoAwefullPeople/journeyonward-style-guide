# Unpack Unpacking Screen

**Route name:** `UnpackUnpacking`
**File:** `src/screens/unpack/UnpackUnpackingScreen.tsx`
**Type:** Stack screen (pushed on top of tabs)

## Purpose
Stage 3b of 4 (final interactive stage): an entirely optional, skippable per-moment tagging step for the highlighted entries from Stage 3a — pick a feeling chip, name the moment, add a note. Then hands off to the Journey Recap player. Serves the Reflective Traveler persona's "capture meaning, not just media" need, but designed to never block completion.

## Entry points & navigation
- From `UnpackHighlightScreen` → `navigate('UnpackUnpacking', { albumId, highlightedIds })`.
- Two exits, both eventually reaching the same place:
  - "Skip" (top-right header link) or the empty-state CTA → `navigateToOutput()` directly.
  - "Done, let's see it →" footer button → saves any entered tags/titles/notes via `updateEntry`, then `navigateToOutput(true)`.
  - `navigateToOutput` itself branches: if this session is mid-*onboarding* (`pendingOnboardingPath === 'unpack'`), it calls `completeOnboarding('unpack')` and then a `CommonActions.reset` to `[MainTabs, JourneyRecapPlayer]` (collapsing history so the user can't navigate back into onboarding); otherwise it's a plain `navigate('JourneyRecapPlayer', { albumId })`, optionally preceded by a 300ms "generating" overlay if the overlay wasn't already showing.
- Back chevron → `navigation.goBack()` (back to `UnpackHighlightScreen`).

## Layout anatomy
1. `MeshGradientBackground` backdrop.
2. Header: back chevron, `UnpackStepBar` (`currentStage={3}` — same stage number as the previous screen, treating Highlight+Unpacking as one visual "stage 3"), and a "Skip" link at the top-right.
3. Title "A little more, if you like" + subtitle explaining everything is optional.
4. Up to `DEFAULT_VISIBLE` (3) `MomentCard`s rendered by default, each showing:
   - Full-width photo banner with a gradient scrim + date/location overlay.
   - "What stood out?" section: a wrapped row of 8 emotion chips (`EMOTION_CHIPS`), each with an emoji + label; selected chips fill with the same 3-color `FLOW_GRADIENT` used in `UnpackHighlightScreen`.
   - "Name this moment · optional" single-line `TextInput`.
   - "Add a note · optional" multiline `TextInput`.
5. "Show N more moments" button when `highlightedEntries.length > DEFAULT_VISIBLE`, expanding to show the rest.
6. Sticky footer: "Done, let's see it →" button (or a spinner while saving).
7. Full-screen "generating" overlay (mesh background + spinner + rotating phrase), shown while saving/transitioning to the recap.
8. Empty-state fallback (`highlightedIds` resolved to zero entries) — just a single CTA button, no cards.

## States
- **Empty**: `highlightedEntries.length === 0` → minimal empty state, single "See your Trip in Review →" button.
- **Populated, partially visible**: `visibleCount < highlightedEntries.length` → "Show N more" button present.
- **Populated, all visible**: after tapping "Show more", or if `highlightedEntries.length <= DEFAULT_VISIBLE` from the start.
- **Saving**: `saving` → footer button shows a spinner and is disabled; `generating` overlay is set true *simultaneously* (not sequentially) so the full-screen transition covers both the save call and the recap hand-off.
- **Generating/transitioning**: `generating` → full-screen overlay fades in (`genOpacity`), with `GENERATING_PHRASES` (3 phrases) cross-fading on a 1800ms/3600ms schedule, mirroring `UnpackProcessingScreen`'s copy-rotation pattern.
- Per-card state (`MomentState`): each highlighted entry has independent `labels: Set<string>`, `title: string`, `note: string`, pre-seeded from the entry's existing `title`/`emotionNote` if any.

## Data & context dependencies
- `useAppContext()` → `entries`, `updateEntry`, `pendingOnboardingPath`, `completeOnboarding`.
- No new entries are created here — only `updateEntry` calls to backfill `emotionLabels`/`title`/`emotionNote` onto entries already created in the previous stage, and only for entries where the user actually entered something (see excerpt).

## Interactions
- Chip tap: toggles that emotion label in the moment's `Set`; selected chips render a gradient fill (via a `LinearGradient` positioned `absoluteFillObject` beneath the label, an unusual toggle-fill technique — see excerpt below in the sibling `UnpackHighlightScreen` doc, same pattern reused here).
- Text inputs: standard controlled inputs, no auto-save — everything is committed only on "Done".
- "Show more": one-way reveal (no "show less").

## Theming notes
- `EMOTION_CHIPS`' emoji/label pairs are fixed regardless of theme; the gradient fill and `COLORS.royalGold`/`babyBlue`/`fern` brand colors are also theme-independent, consistent with the rest of the Unpack flow's "flow gradient" branding.
- Card surface/border/text follow `useTheme()`.

## Reference UI skeleton
Whole render tree, business logic stripped. `EMOTION_CHIPS` = 8 `{label, stored, emoji}` entries (display label → stored vocabulary value). `DEFAULT_VISIBLE = 3`. `FLOW_GRADIENT` shared with `UnpackHighlightScreen`.

```tsx
// source: src/screens/unpack/UnpackUnpackingScreen.tsx (skeleton — business logic stripped)

// ── MomentCard sub-component ────────────────────────────────────────────────
const MomentCard = ({ entry, state, onToggleLabel, onChangeTitle, onChangeNote }) => (
  <View style={cardStyles.container}> {/* borderRadius:16, border, overflow:hidden */}
    <View style={cardStyles.banner}> {/* height: BANNER_H = 160 */}
      {photoUri ? <Image source={{ uri: photoUri }} style={cardStyles.bannerImg} /> : <View style={[cardStyles.bannerImg, cardStyles.bannerPlaceholder]}><Ionicons name="image-outline" size={32} /></View>}
      <LinearGradient colors={['transparent', 'rgba(0,0,0,0.62)']} style={cardStyles.bannerScrim}>
        <AppText style={cardStyles.bannerDate}>{formattedDate}</AppText>
        {location && <AppText style={cardStyles.bannerLocation}>{location}</AppText>}
      </LinearGradient>
    </View>

    <View style={cardStyles.section}>
      <AppText style={cardStyles.sectionLabel}>What stood out?</AppText>
      <View style={cardStyles.chipWrap}> {/* row, flexWrap, gap: SPACING.xs+2 */}
        {EMOTION_CHIPS.map(chip => {
          const selected = state.labels.has(chip.stored);
          return (
            <TouchableOpacity key={chip.stored} onPress={() => onToggleLabel(chip.stored)} style={[chipStyles.chip, !selected && { backgroundColor: colors.surfaceVariant, borderColor: colors.border }]}>
              {selected && <LinearGradient colors={FLOW_GRADIENT} start={{x:0,y:.5}} end={{x:1,y:.5}} style={StyleSheet.absoluteFillObject} />} {/* gradient toggle-fill, see "Key technique" */}
              <AppText style={chipStyles.emoji}>{chip.emoji}</AppText>
              <AppText style={[chipStyles.label, { color: selected ? COLORS.black : colors.textPrimary }]}>{chip.label}</AppText>
            </TouchableOpacity>
          );
        })}
      </View>
    </View>

    <View style={cardStyles.section}>
      <AppText style={cardStyles.sectionLabel}>Name this moment · optional</AppText>
      <TextInput style={cardStyles.lineInput} value={state.title} onChangeText={onChangeTitle} maxLength={80} />
    </View>

    <View style={[cardStyles.section, cardStyles.lastSection]}>
      <AppText style={cardStyles.sectionLabel}>Add a note · optional</AppText>
      <TextInput style={[cardStyles.lineInput, cardStyles.noteInput]} value={state.note} onChangeText={onChangeNote} multiline />
    </View>
  </View>
);

// ── Screen ───────────────────────────────────────────────────────────────────
return (
  <View style={[styles.container, { backgroundColor: colors.background }]}>
    <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />

    <View style={[styles.header, { paddingTop: insets.top + SPACING.sm }]}>
      <View style={styles.headerRow}> {/* row, justify-between */}
        <TouchableOpacity style={styles.backBtn} onPress={() => navigation.goBack()}><Ionicons name="chevron-back" size={22} /></TouchableOpacity>
        <TouchableOpacity onPress={navigateToOutput} style={styles.skipBtn}><AppText style={styles.skipText}>Skip</AppText></TouchableOpacity>
      </View>
      <UnpackStepBar currentStage={3} />
    </View>

    {isEmpty ? (
      <View style={styles.emptyState}> {/* flex:1, justify-center */}
        <TouchableOpacity style={styles.outputBtn} onPress={navigateToOutput}><AppText style={styles.outputBtnText}>See your Trip in Review →</AppText></TouchableOpacity>
      </View>
    ) : (
      <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === 'ios' ? 'padding' : 'height'}>
        <ScrollView contentContainerStyle={[styles.scrollContent, { paddingBottom: insets.bottom + 100 }]} keyboardShouldPersistTaps="handled">
          <View style={styles.titleSection}>
            <AppText style={styles.screenTitle}>A little more, if you like</AppText> {/* Playfair Display italic bold, 26px */}
            <AppText style={styles.screenSubtitle}>Tag what stood out, name a moment, add a note. All optional — skip anytime.</AppText>
          </View>

          {visibleEntries.map(entry => (
            <MomentCard key={entry.id} entry={entry} state={momentStates.get(entry.id)} onToggleLabel={...} onChangeTitle={...} onChangeNote={...} />
          ))}

          {hiddenCount > 0 && (
            <TouchableOpacity style={styles.showMoreBtn} onPress={() => setVisibleCount(highlightedEntries.length)}>
              <AppText style={styles.showMoreText}>Show {hiddenCount} more moment{s}</AppText>
              <Ionicons name="chevron-down" size={14} />
            </TouchableOpacity>
          )}
        </ScrollView>
      </KeyboardAvoidingView>
    )}

    {!isEmpty && (
      <View style={styles.footer}> {/* absolute bottom, border-top, bg: colors.background */}
        <TouchableOpacity style={[styles.doneBtn, { opacity: saving ? 0.7 : 1 }]} onPress={handleDone} disabled={saving}>
          {saving ? <ActivityIndicator size="small" /> : <AppText style={styles.doneBtnText}>Done, let's see it →</AppText>}
        </TouchableOpacity>
      </View>
    )}

    {generating && (
      <Animated.View style={[StyleSheet.absoluteFill, styles.generatingOverlay, { opacity: genOpacity }]} pointerEvents="box-only">
        <MeshGradientBackground baseColor={colors.background} lightMode={!isDark} />
        <ActivityIndicator size="large" color={colors.textSecondary} />
        <Animated.Text style={[styles.generatingText, { opacity: phraseOpacity }]}>{GENERATING_PHRASES[phraseIndex]}</Animated.Text>
        {/* GENERATING_PHRASES cross-fade on 1800ms/3600ms, mirroring UnpackProcessingScreen's copy rotation */}
      </Animated.View>
    )}
  </View>
);
```

**Key technique(s):**
- The save step (`handleDone`) only writes fields the user actually touched — a partial `updateEntry` payload per entry (`...(hasLabels && {...})`) rather than always overwriting all three fields with possibly-empty values — `file:400-430`.
- Selected emotion chips render a `LinearGradient` positioned `absoluteFillObject` *beneath* the label as the "selected" fill, rather than a plain background-color swap — an unusual toggle-fill technique shared with `UnpackHighlightScreen`'s highlight strip.

## Screenshot
_Not yet captured. TODO: capture light + dark for: default view (≤3 cards, "Show more" visible if applicable), a card with chips/title/note filled in, the "generating" overlay, and the empty state._

## Open questions / follow-ups
- The `EMOTION_CHIPS` display labels ("Awe", "Connection", "Challenge", "Surprise", "Peaceful", "Funny", "Comfort", "Adventure") are a *different, smaller vocabulary* than the PRD's 25-emotion Russell's-circumplex grid used elsewhere in the app (`EmotionCaptureScreen`) — each chip maps to one specific stored label from that larger vocabulary (e.g. "Awe" → stored `"Amazed"`, "Comfort" → stored `"Joyful"`) rather than exposing the full grid. Worth confirming this simplified chip set is an intentional UX choice for this specific low-friction context, not an oversight of the richer emotion-capture system.
- This screen and `UnpackTripDetailsScreen`/`UnpackHighlightScreen` all reuse the identical `FLOW_GRADIENT` constant (`[COLORS.babyBlue, COLORS.royalGold, COLORS.fern]`) independently defined in each file — a candidate for extraction into a shared constant if a fourth Unpack-flow screen is ever added.
- The onboarding hand-off branch (`pendingOnboardingPath === 'unpack'`) means this screen's behavior differs meaningfully depending on whether the user is unpacking their very first trip during onboarding vs. unpacking a later trip from the "+" menu — worth cross-referencing with the onboarding screens' docs when those are written.
