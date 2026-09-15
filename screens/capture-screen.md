# Capture

**Route name:** `Capture`
**File:** `src/screens/CaptureScreen.tsx`
**Type:** Modal (`presentation: 'modal'`, `gestureEnabled: false`)

## Purpose
The live-capture camera: BeReal/Instagram-style embedded camera that accumulates up to 3 photos before handing off to an inline review step. Serves the Live-Capture Traveler persona directly — this is the low-friction "point and shoot" path from the PRD's Flow 5.

## Entry points & navigation
- Entry: the plus-menu "Live Capture" tile in `AppNavigator.tsx`'s `MainTabs` (`handlePlusMenuAction('Capture')`), and `HomeScreen`'s floating camera button (`navigation.navigate('Capture')`).
- No `route.params`.
- Exit:
  - Back arrow → `navigation.goBack()`, with a "Discard photos?" confirmation `Alert` if `capturedPhotos.length > 0`.
  - After 3 photos are captured (auto) or "Next →" is tapped (manual, once ≥1 photo exists) → internal `phase` state flips to `'review'`, rendering `CaptureReviewView` (`src/components/capture/CaptureReviewView.tsx`) in place — no navigation transition, just a component swap within the same screen.
  - From the review view's `onSaved(entryId, previewFirstPhotoUri)` callback → `navigation.reset({ index: 1, routes: [{ name: 'MainTabs' }, { name: 'CaptureSaved', params: {...} }] })`, i.e. the whole modal stack is replaced so back-navigation from `CaptureSaved` lands on `MainTabs`, not back into the camera.

## Layout anatomy
1. Back arrow, top-left, absolutely positioned over everything (`zIndex: 100`).
2. Camera preview (`CameraView` from `expo-camera`, lazy-loaded via dynamic `import()`), pinch-to-zoom wrapped in a `PinchGestureHandler`, positioned with hand-tuned absolute offsets scaled by `scaleFactor` (device-width-relative, base 390px).
3. Camera controls overlay (bottom of the camera frame): flash toggle, zoom-preset pill (shows current effective zoom, e.g. "1x"/"0.5x"), front/back camera switch.
4. Bottom bar below the camera frame: left slot = photo count badge (only visible once ≥1 photo captured), center = circular shutter button, right slot = "Next →" text button (only visible once ≥1 photo captured).

## States
- **`!cameraReady`** (native module not yet dynamically imported) → "Loading camera..." text only.
- **`!permissionGranted`** → "Camera permission required" text + "Grant Permission" button (re-requests via `Camera.requestCameraPermissionsAsync()`).
- **`phase === 'review'`** → entire screen replaced by `<CaptureReviewView photos={capturedPhotos} onBack={...} onSaved={...} />` (a separate component, not covered by this doc).
- **Camera mounting** (`CameraView` module loaded but `!cameraHardwareReady`) → placeholder view: "Preparing camera…" or, if `onMountError` fired, "Camera unavailable" + the raw error message.
- **`atLimit`** (`capturedPhotos.length >= MAX_PHOTOS` = 3): shutter capture is a no-op (`handleCapture` early-returns); reaching the limit via capture auto-advances to `phase: 'review'`.
- **`isPinching`**: disables the zoom-preset button (`disabled={!cameraHardwareReady || isPinching}`) so a preset tap can't fight an in-progress pinch gesture.
- Photo count badge and "Next →" button both conditionally render only when `hasPhotos` is true — absent entirely (not just disabled/hidden) at 0 photos.

## Data & context dependencies
- No `AppContext` reads — purely local component state (camera/zoom/lens state, `capturedPhotos` array, `phase`).
- `useTheme()` → `colors` (passed into `createStyles(colors)`, a factory pattern used consistently across screens with theme-dependent `StyleSheet.create` calls).
- Dynamic `import('expo-camera')` for `CameraView`/`Camera` — deferred exactly like the app's native-navigation-module pattern described in `CLAUDE.md`, but here it's for a heavy native camera module rather than React Navigation.

## Interactions
- `PinchGestureHandler` (react-native-gesture-handler) drives zoom: pinch delta is scaled by `ZOOM_SENSITIVITY` (0.35) with a dead zone (`PINCH_DEAD_ZONE = 0.015`) and throttled to one update per 16ms (`PINCH_UPDATE_INTERVAL_MS`).
- iOS-specific lens-switching logic: pinching below `ZOOM_MIN` while on the physical wide lens auto-switches to the ultra-wide lens (if available) and resets zoom to 0, and the reverse when pinching an ultra-wide lens above `ZOOM_MAX` — i.e. the pinch gesture transparently crosses lens boundaries rather than just scaling one lens's digital zoom.
- Shutter button: legacy `Animated` press feedback (`scale → 0.92` on press-in, spring back to 1 on press-out via `createSpringConfig()`), guarded against re-entrant captures with `isCapturingRef`.
- `takePictureAsync({ quality: 0.8, base64: false })` — photos accumulate in `capturedPhotos` (local URIs); the array is only handed off to `CaptureReviewView`, not persisted until the review step saves.

## Theming notes
Theme-aware via `useTheme()` throughout (`colors.textPrimary`, `colors.stone600` camera placeholder background, `colors.borderLight`, `colors.zinc300` shutter button, `colors.buttonGray` permission button) — camera control chips themselves use a fixed `rgba(0,0,0,0.5)` scrim regardless of theme (they sit over live camera video, not app UI, so theme-following would look wrong).

## Reference UI skeleton
Full render tree for the camera phase (the `review` phase hands off entirely to `CaptureReviewView`, out of scope here). Absolute-position offsets are hand-tuned Figma pixel values scaled by `scaleFactor`:

```tsx
// source: src/screens/CaptureScreen.tsx (skeleton — business logic stripped, see cited lines for real implementation)
const MAX_PHOTOS = 3;

const CaptureScreen = () => {
  const styles = useMemo(() => createStyles(colors), [colors]);
  // camera/zoom/lens/facing state, capturedPhotos[], phase ('camera' | 'review') — see :48-355

  if (!cameraReady) return <View style={styles.container}><Text style={styles.loadingText}>Loading camera...</Text></View>;

  if (!permissionGranted) {
    return (
      <View style={styles.container}>
        <Text style={styles.loadingText}>Camera permission required</Text>
        <TouchableOpacity style={styles.permissionButton} onPress={/* requestCameraPermissionsAsync — :371-379 */ requestPermission}>
          <Text style={styles.permissionButtonText}>Grant Permission</Text>
        </TouchableOpacity>
      </View>
    );
  }

  if (phase === 'review') {
    return <CaptureReviewView photos={capturedPhotos} onBack={/* :390-394 */ onBack} onSaved={/* navigation.reset to MainTabs + CaptureSaved — :395-406 */ onSaved} />;
  }

  const imageStyle = { width: windowWidth, height: 463.033 * scaleFactor, left: 0, top: 107.07 * scaleFactor - containerTop };
  const buttonSize = DIMENSIONS.captureButtonSize * scaleFactor;
  const captureButtonTop = 644 * scaleFactor - containerTop;

  return (
    <View style={styles.container}>
      <TouchableOpacity style={[styles.backButton, { top: insets.top + SPACING.sm, left: 18 * scaleFactor }]} onPress={handleBack}>
        <BackArrow width={23.75 * scaleFactor} height={26.25 * scaleFactor} color={colors.textPrimary} />
      </TouchableOpacity>

      <View style={[styles.contentContainer, { top: containerTop }]}>
        <Animated.View style={[styles.cameraContainer, imageStyle]}>
          {CameraView ? (
            <>
              <PinchGestureHandler enabled={permissionGranted} onGestureEvent={handlePinchGesture} onHandlerStateChange={handlePinchStateChange}>
                <View style={styles.camera}>
                  <CameraView
                    key={`camera-${facing}`}
                    style={styles.camera}
                    facing={facing}
                    flash={flashMode}
                    zoom={cameraZoom}
                    selectedLens={selectedLens}
                    onCameraReady={/* getAvailableLensesAsync — :461-469 */ onCameraReady}
                    onMountError={/* sets cameraMountError — :471-476 */ onMountError}
                  />
                </View>
              </PinchGestureHandler>

              {/* Camera controls overlay — bottom of frame */}
              <View style={styles.cameraControlsOverlay} pointerEvents="box-none">
                <View style={styles.cameraControlsRow}>
                  <TouchableOpacity style={styles.cameraControlButton} onPress={() => setFlashMode(m => m === 'on' ? 'off' : 'on')}>
                    <FlashIcon color={flashMode === 'on' ? '#DEC337' : colors.textPrimary} />
                  </TouchableOpacity>
                  <TouchableOpacity style={styles.zoomToggleButton} onPress={handleZoomButtonPress} disabled={!cameraHardwareReady || isPinching}>
                    <Text style={styles.zoomToggleText}>{zoomButtonLabel /* e.g. "1x" */}</Text>
                  </TouchableOpacity>
                  <TouchableOpacity style={styles.cameraControlButton} onPress={() => setFacing(f => f === 'back' ? 'front' : 'back')}>
                    <CameraSwitchIcon color={colors.textPrimary} />
                  </TouchableOpacity>
                </View>
              </View>
            </>
          ) : (
            <View style={styles.cameraPlaceholder}>
              <Text style={styles.placeholderText}>{cameraMountError ? 'Camera unavailable' : 'Preparing camera…'}</Text>
              {cameraMountError && <Text style={styles.cameraErrorText}>{cameraMountError}</Text>}
            </View>
          )}
        </Animated.View>

        {/* Bottom bar: [count badge] [shutter] [Next →] — count badge & Next both absent (not just hidden) at 0 photos */}
        <View style={[styles.bottomBar, { top: captureButtonTop }]}>
          <View style={styles.bottomBarSide}>
            {hasPhotos && <View style={styles.countBadge}><Text style={styles.countBadgeText}>{capturedPhotos.length}</Text></View>}
          </View>

          <Animated.View style={{ transform: [{ scale: captureButtonScale }] }}>
            <TouchableOpacity
              style={[styles.captureButton, { width: buttonSize, height: buttonSize, borderRadius: buttonSize / 2 }]}
              onPress={handleCapture}
              onPressIn={handleCapturePressIn}
              onPressOut={handleCapturePressOut}
            >
              <View style={[styles.captureButtonInner, { width: innerSize, height: innerSize, borderRadius: innerSize / 2 }]} />
            </TouchableOpacity>
          </Animated.View>

          <View style={styles.bottomBarSide}>
            {hasPhotos && (
              <TouchableOpacity style={styles.nextButton} onPress={handleNext}>
                <Text style={styles.nextButtonText}>Next →</Text>
              </TouchableOpacity>
            )}
          </View>
        </View>
      </View>
    </View>
  );
};

function createStyles(colors) {
  return StyleSheet.create({
    container: { flex: 1, backgroundColor: colors.background },
    contentContainer: { position: 'absolute', left: 0, right: 0, bottom: 0 },
    backButton: { position: 'absolute', zIndex: 100, justifyContent: 'center', alignItems: 'center' },
    cameraContainer: { position: 'absolute', backgroundColor: colors.stone600, borderRadius: 7, borderWidth: 2, borderColor: colors.borderLight, overflow: 'hidden' },
    camera: { width: '100%', height: '100%' },
    cameraControlsOverlay: { position: 'absolute', bottom: 0, left: 0, right: 0, justifyContent: 'flex-end' },
    cameraControlsRow: { flexDirection: 'row', alignItems: 'center', alignSelf: 'center', gap: 8 },
    cameraControlButton: { backgroundColor: 'rgba(0,0,0,0.5)', alignItems: 'center', justifyContent: 'center' }, // fixed scrim regardless of theme — sits over live video
    zoomToggleButton: { minWidth: 58, paddingHorizontal: 10, paddingVertical: 7, borderRadius: 16, backgroundColor: 'rgba(0,0,0,0.5)' },
    bottomBar: { position: 'absolute', left: 0, right: 0, flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', paddingHorizontal: 32 },
    bottomBarSide: { width: 74, alignItems: 'center', justifyContent: 'center' },
    captureButton: { backgroundColor: colors.zinc300, justifyContent: 'center', alignItems: 'center', borderWidth: 0.5, borderColor: colors.borderLight },
    captureButtonInner: { backgroundColor: colors.zinc300, borderWidth: 1, borderColor: colors.background },
    countBadge: { width: 44, height: 44, borderRadius: 10, backgroundColor: 'rgba(255,255,255,0.15)', borderWidth: 1, borderColor: colors.borderLight, alignItems: 'center', justifyContent: 'center' },
  });
}
```

**Key technique(s):** the iOS lens-crossing zoom logic (`:198-220`) — pinching below `ZOOM_MIN` on the physical wide lens auto-switches to the ultra-wide lens and resets zoom to 0 (and the reverse above `ZOOM_MAX` on ultra-wide) — is the least obvious behavior here; the pinch gesture transparently crosses lens boundaries rather than just scaling one lens's digital zoom. Camera control chips deliberately use a fixed `rgba(0,0,0,0.5)` scrim (not `useTheme()`) since they sit over live camera video, not app UI.

## Screenshot
_Not yet captured. TODO: capture light + dark: camera live view with 0/1/3 photos captured (count badge + Next button visibility), permission-denied state, camera-unavailable/mount-error state._

## Open questions / follow-ups
- The PRD (Flow 5 / F5.1–F5.4) describes a Photo/Video/Audio type selector on this screen ("embedded camera for photos automatically opens, but user can change media type"); this implementation only captures photos (up to 3) — video and audio capture aren't present in `CaptureScreen.tsx`. Worth confirming with product whether that's intentionally descoped for the current build or lives elsewhere (e.g. `AudioWaveform.tsx`/`EmotionalCaptureFlow.tsx` exist in `src/components/capture/` per `CLAUDE.md` but this screen doesn't reference them).
- `CaptureReviewView` (the component this screen hands off to at `phase: 'review'`) is out of scope for this doc but is where titles, descriptions, and the actual `createEntry`/save call almost certainly happen — worth its own doc if it's complex enough.
