# Screens

Two layers of reference live in this folder:

1. **Per-screen docs** (below, one `.md` per screen) — written 2026-09-15 by reading the live `Journey Onward` codebase screen-by-screen. Each follows `TEMPLATE.md`: purpose, entry points/nav, layout anatomy, states, data/context dependencies, interactions, theming notes, a real code excerpt with a `file:line` citation, and open questions. These describe **actual current behavior**, not the PRD's original intent — see "Known gaps vs. PRD" below for where the two diverge.
2. **Screenshots** (`.png`) — visual captures from the running v1 app (iOS Simulator, real seeded data), 2026-09-14. Only 6 exist; most screens still need one. Pair a screenshot with its doc — the screenshot shows the result, the doc explains the structure and cites the code.

## Per-screen docs

### Onboarding
| Doc | Route | Type |
|---|---|---|
| [onboarding-path-screen.md](onboarding-path-screen.md) | `OnboardingPathScreen` | Stack |
| [onboarding-slides-screen.md](onboarding-slides-screen.md) | `OnboardingSlidesScreen` | Stack |

### Home / Studio / Capture / Entries / Journals
| Doc | Route | Type |
|---|---|---|
| [home-screen.md](home-screen.md) | `HomeScreen` | Tab |
| [capture-screen.md](capture-screen.md) | `CaptureScreen` | Modal |
| [capture-saved-screen.md](capture-saved-screen.md) | `CaptureSavedScreen` | Stack |
| [entry-detail-screen.md](entry-detail-screen.md) | `EntryDetailScreen` | Stack |
| [entry-edit-screen.md](entry-edit-screen.md) | `EntryEditScreen` | Stack |
| [journal-detail-screen.md](journal-detail-screen.md) | `JournalDetailScreen` | Stack |
| [notes-editor-screen.md](notes-editor-screen.md) | `NotesEditorScreen` | Transparent modal |
| [emotion-capture-screen.md](emotion-capture-screen.md) | `EmotionCaptureScreen` | Stack |

### Trip organization & review
| Doc | Route | Type |
|---|---|---|
| [trip-album-create-screen.md](trip-album-create-screen.md) | `TripAlbumCreateScreen` | Stack |
| [trip-album-detail-screen.md](trip-album-detail-screen.md) | `TripAlbumDetailScreen` | Stack |
| [suggested-albums-screen.md](suggested-albums-screen.md) | `SuggestedAlbumsScreen` | Stack |
| [add-past-experiences-screen.md](add-past-experiences-screen.md) | `AddPastExperiencesScreen` | Stack |
| [import-latest-screen.md](import-latest-screen.md) | `ImportLatestScreen` | Stack |
| [review-screen.md](review-screen.md) | `ReviewScreen` | Tab |
| [settings-screen.md](settings-screen.md) | `SettingsScreen` | Stack |

### Journey Recap & stats
| Doc | Route | Type |
|---|---|---|
| [journey-recap-candidates-screen.md](journey-recap-candidates-screen.md) | `JourneyRecapCandidatesScreen` | Stack |
| [journey-recap-player-screen.md](journey-recap-player-screen.md) | `JourneyRecapPlayerScreen` | Stack |
| [journey-recap-post-screen.md](journey-recap-post-screen.md) | `JourneyRecapPostScreen` | Stack |
| [memories-slideshow-screen.md](memories-slideshow-screen.md) | `MemoriesSlideshowScreen` | Stack |
| [countries-visited-map-screen.md](countries-visited-map-screen.md) | `CountriesVisitedMapScreen` | Stack |
| [country-ranking-full-list-screen.md](country-ranking-full-list-screen.md) | `CountryRankingFullListScreen` | Stack |
| [full-country-calendar-screen.md](full-country-calendar-screen.md) | `FullCountryCalendarScreen` | Stack |
| [day-detail-screen.md](day-detail-screen.md) | `DayDetailScreen` | Modal |

### Postcards
| Doc | Route | Type |
|---|---|---|
| [postcard-templates-screen.md](postcard-templates-screen.md) | `PostcardTemplates` | Stack |
| [postcard-editor-screen.md](postcard-editor-screen.md) | `PostcardEditor` | Stack |
| [postcard-story-editor-screen.md](postcard-story-editor-screen.md) | `PostcardStoryEditor` | Stack |
| [saved-postcards-screen.md](saved-postcards-screen.md) | `SavedPostcards` | Stack |

### Unpack a Trip (wizard)
| Doc | Route | Type |
|---|---|---|
| [unpack-trip-details-screen.md](unpack-trip-details-screen.md) | `UnpackTripDetailsScreen` | Stack |
| [unpack-add-memories-screen.md](unpack-add-memories-screen.md) | `UnpackAddMemoriesScreen` | Stack |
| [unpack-processing-screen.md](unpack-processing-screen.md) | `UnpackProcessingScreen` | Stack |
| [unpack-highlight-screen.md](unpack-highlight-screen.md) | `UnpackHighlightScreen` | Stack |
| [unpack-unpacking-screen.md](unpack-unpacking-screen.md) | `UnpackUnpackingScreen` | Stack |

## Screenshots

| File | Screen | Theme | Notes |
|---|---|---|---|
| `home-feed-list.png` | Home — single-column feed mode | Light | `FeedStyleEntryCard`: full-bleed photo, gradient scrims, country/date/title overlay. See `patterns/cards-and-feed-grid.md` and `home-screen.md`. |
| `home-grid-pinterest.png` | Home — Pinterest masonry grid mode | Light | Two-column masonry; also shows a **recap card** with the gradient border treatment. See `patterns/cards-and-feed-grid.md` and `home-screen.md`. |
| `home-grid-dark.png` | Home — Pinterest masonry grid mode | Dark | Same layout as above under the dark palette — useful for comparing how card contrast/scrims hold up in both themes. |
| `capture-action-sheet.png` | Capture bottom sheet (opened via the `+` tab) | Light | 2×2 action grid (Live Capture / Import Memories / Create A Postcard / Journal Entry) + a gradient-bordered "Unpack A Trip" promo card. See `capture-screen.md`. |
| `review-grid-filters.png` | Review — All Memories grid, with search + filter chips | Light | Segmented view control (stats / trips / grid) at top, search bar, filter chips (Experiences / Bookmarks / Photos / Notes) above the masonry grid. See `review-screen.md`. |
| `review-trips-dark.png` | Review — Your Trips tab (Recaps & Stories carousel + Albums) | Dark | Horizontal carousel of gradient-bordered recap/story cards, with a trip-albums grid below (empty-state albums show a muted "No photos yet" placeholder). See `review-screen.md`. |

### Not yet captured

Every screen above except Home, Capture (action sheet only), and Review has zero screenshot coverage — 28 of 34. Highest-value next captures: `HomeScreen`'s Travel Mode card / active-trip header / long-press overlay (none of the 3 existing Home screenshots show these), the full Capture photo flow (not just the action sheet), the Emotion Capture emoji grid, the Journey Recap Player slideshow, and the Unpack wizard's 5 steps. Capture light + dark for each.

## Known gaps vs. PRD / context docs

The per-screen docs above describe what the code actually does, which in several places has moved on from `PRD_JourneyOn_MVP.md` and the context doc. Flagging here so nobody assumes the PRD is current:

- **Capture is photo-only.** `CaptureScreen` supports up to 3 photos with pinch-zoom lens switching — no video or audio capture, despite the PRD's Photo/Video/Audio selector (Flow 5).
- **Emotion capture uses a different model.** The PRD specifies valence/arousal sliders mapped to a 25-emotion Russell's-Circumplex grid; the live `EmotionCaptureScreen`/`EmotionalCaptureFlow` ships a fixed 3×3 emoji grid + free-text note instead.
- **Trip album creation dropped fields.** `TripAlbumCreateScreen` no longer has a location field or the pre-trip chatbot (Plans/Intentions/Travel Reasons) — it's just title → start date → end date → description.
- **Review doesn't match its PRD flow.** It's a 3-way Stats / Your Trips / Gallery segmented control, not the grid/map toggle with trip/time/location/draft filters described in Flow 7.
- **Two entire features postdate the PRD and aren't mentioned in it at all:** the 4-screen Postcards feature and the 5-screen "Unpack a Trip" wizard.

## Other things worth a look (not fixed, just noted)

- `SavedPostcardsScreen` is registered in the navigator but nothing currently navigates to it — orphaned.
- `JourneyRecapCandidatesScreen` is fully built but no live path reaches it either; every current entry point jumps straight to the Player with a known album.
- `SettingsScreen.tsx:407` has a `// DEV — remove before TestFlight` preview-onboarding row still present.
- `JournalDetailScreen`'s Edit menu option is a dead end (`Alert.alert('Not available', ...)`).
- Several screens hand-roll near-identical logic independently rather than sharing it: `EntryDetailScreen`/`JournalDetailScreen`'s action-menu animation, `JourneyRecapPlayerScreen`/`MemoriesSlideshowScreen`'s cross-fade/progress-bar timing, and three unrelated postcard-style template pickers (`MemoriesSlideshowScreen`, `JourneyRecapPostScreen`, `PostcardTemplatesScreen`).
- `DayDetailScreen` and `FullCountryCalendarScreen` each independently resolve "what country was this day" rather than sharing one util — risk of disagreeing for the same day.

Individual screen docs have more detail and code citations for each of these under their own "Open questions / follow-ups" section.
