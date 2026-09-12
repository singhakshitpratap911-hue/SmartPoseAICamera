# SmartPose AI Camera

A standalone native Android app (Kotlin, Android 14 / SDK 34) with a CameraX viewfinder and a
Huawei-style **AI Pose Guide engine** powered by on-device Google MediaPipe pose tracking.

![Stack](https://img.shields.io/badge/Kotlin-2.0-7F52FF) ![Compose](https://img.shields.io/badge/Jetpack%20Compose-Material3-4285F4) ![CameraX](https://img.shields.io/badge/CameraX-1.3.4-3DDC84) ![MediaPipe](https://img.shields.io/badge/MediaPipe-Tasks%20Vision-00C4B3)

## Features

### Camera pipeline (CameraX)
- Full viewfinder, **tap-to-focus** (AF+AE metering), **pinch-to-zoom**, double-tap to flip lens
- **Exposure compensation slider** (EV), torch/flash toggle, HDR toggle
- **Aspect ratios**: 4:3 / 16:9 native, 1:1 via capture-time center crop
- Photo capture with white-flash shutter animation + live **gallery thumbnail**
- Scoped-storage saves to `Pictures/SmartPoseAI` (no permissions needed on API 29+)

### Capture modes
| Mode | Behavior |
|---|---|
| **Photo** | Clean viewfinder |
| **Portrait** | Software depth-effect (gradient-masked progressive background blur) |
| **AI Pose** | Ghost silhouette + real-time alignment coaching |
| **Pro** | Rule-of-thirds grid, eye-line marker, horizon leveler, live luminance histogram |

### AI Pose Guide engine (on-device, <30 ms/frame)
- **MediaPipe Pose Landmarker (lite)** runs in `LIVE_STREAM` mode with a GPU delegate inside the
  CameraX `ImageAnalysis` stream (KEEP_ONLY_LATEST back-pressure = zero pipeline stalls)
- **Scene understanding** (pure geometry, <1 ms): Standing / Sitting / Squatting / Leaning,
  Full-body / Half-body / Close-up, rule-of-thirds score, eye-line height, balance offset,
  shoulder tilt
- **Ghost overlay**: semi-transparent wireframe of the target pose rendered on the viewfinder
- **Alignment cues**: per-joint Red → Yellow → Green coloring + correction arrows for misplaced
  joints; **green glow + double-buzz haptic** when similarity ≥ 85%
- **Matching**: hip-centered, torso-normalized landmarks compared with 65% cosine similarity +
  35% joint-angle agreement (8 angle triplets) — invariant to distance and body size
- **Pose carousel**: 10 built-in reference poses (Casual, Professional, Creative, Sitting, Candid)
  with skeleton previews, manual selection or scene-driven **Auto-match**

## Architecture (MVVM + Clean layers)

```
┌─ ui/           Compose screens & overlays (View)
│   CameraScreen, CameraPreview, PoseOverlay,
│   CompositionGuideOverlay, PoseSelectorBottomSheet
├─ ui/camera/    CameraViewModel (ViewModel) — single StateFlow<CameraUiState>
├─ camera/       CameraController (CameraX use-cases), PortraitDepthEffect
├─ pose/         PoseDetectorAnalyzer (MediaPipe), SceneClassifier, PoseMatcher
└─ data/         PoseLibrary (asset loader), model/ (domain entities, topology)
```

Data flow: `ImageProxy → Analyzer → PoseFrame → ViewModel.match() → StateFlow → Compose recomposition`.
Inference is asynchronous (detectAsync), so the camera pipeline never blocks on ML.

## Setup

1. **Android Studio** Hedgehog (2023.1.1) or newer, JDK 17.
2. Clone / unzip this project and open the root folder.
3. **Download the pose model** into `app/src/main/assets/`:
   ```
   curl -L -o app/src/main/assets/pose_landmarker_lite.task \
     https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_lite/float16/latest/pose_landmarker_lite.task
   ```
   (The app runs without it — AI features simply report "model missing".)
4. Sync Gradle, then **Run** on a physical device (pose inference needs a real camera;
   emulators work but performance varies).

Gradle wrapper note: if `gradle/wrapper/gradle-wrapper.jar` is not present, run
`gradle wrapper --gradle-version 8.9` once with a local Gradle install, or let
Android Studio regenerate it on first sync (File → Sync Project).

## Permissions
`CAMERA`, `RECORD_AUDIO`, `VIBRATE`, `READ_MEDIA_IMAGES` (≤ API 32: `READ_EXTERNAL_STORAGE`).
Portrait lock and edge-to-edge are set in the manifest/activity.

## Extending
- Add poses: append an entry to `app/src/main/assets/reference_poses.json` (13 anchored joints;
  the loader derives the remaining 20 MediaPipe landmarks automatically).
- Higher accuracy: swap the lite model for `pose_landmarker_full.task` / `heavy` and update
  `MODEL_FILE` in `PoseDetectorAnalyzer`.
- True depth portrait: replace `PortraitDepthEffect` with a MediaPipe selfie-segmentation mask.
"# SmartPoseAICamera" 
