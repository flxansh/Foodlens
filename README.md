# FoodLens

A cross-platform (Android and iOS) Flutter app that tells you what is in your food.
Two ways to scan, one results screen:

- **Label mode** reads the ingredient list on a package with on-device OCR (Google ML Kit). It works offline.
- **Food mode** sends a photo of a dish to a small backend, where a vision model names the food and lists its likely ingredients.
- **Auto** looks for label text first and falls back to food recognition.

Results show a color-coded ingredient strip (fine, watch, avoid), allergen and diet alerts based on
your own profile, additive notes, "may contain" warnings from the label, a rough score, and scan history.

> **Status:** this was written without a Flutter SDK available, so it has not been compiled.
> The label parser and the allergen, diet and additive logic were checked against a line-by-line
> Python port. Expect to fix a small compile or version issue on your first `flutter analyze`.

## 1. Run the backend

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export ANTHROPIC_API_KEY=your-key-here
uvicorn main:app --host 0.0.0.0 --port 8000
```

Check it: open http://localhost:8000/health. Options (`APP_KEY`, `ANTHROPIC_MODEL`) are in `backend/.env.example`.

## 2. Run the app

```bash
# in the project root (where pubspec.yaml is)
flutter create --project-name foodlens .   # generates android/ and ios/, keeps lib/ and pubspec.yaml
flutter pub get
flutter run --dart-define=API_BASE_URL=http://YOUR_COMPUTER_IP:8000
```

Use your computer's LAN IP for a real phone. The Android emulator default (`http://10.0.2.2:8000`)
and the iOS simulator (`http://localhost:8000`) need no flag or a different one.

### Platform setup (one time)

**Android**
- `android/app/build.gradle(.kts)`: make sure `minSdk` is 21 or higher.
- `android/app/src/main/AndroidManifest.xml`: add
  `<uses-permission android:name="android.permission.INTERNET"/>` and
  `<uses-permission android:name="android.permission.CAMERA"/>`.
  For local development over plain http, also add `android:usesCleartextTraffic="true"` to `<application>`.
  Remove that once your backend is on https.

**iOS**
- `ios/Podfile`: set `platform :ios, '15.5'`.
- `ios/Runner/Info.plist`: add `NSCameraUsageDescription` ("FoodLens uses the camera to scan labels and food")
  and `NSPhotoLibraryUsageDescription` ("Choose a photo to scan").
- Test scanning on a real iPhone. ML Kit has had trouble on Apple Silicon simulators.

## How it fits together

```
lib/
  main.dart, theme.dart, config.dart
  models/      ScanEntry (a saved scan), Analysis (findings)
  data/        knowledge.dart: allergen and diet keywords, additive notes
  services/
    label_scanner.dart         ML Kit OCR
    ingredient_parser.dart     OCR text -> clean ingredient list (+ "may contain")
    ingredient_analyzer.dart   ingredients -> allergens, diets, additives, score
    food_vision_service.dart   photo -> backend -> dish + ingredients
    scan_pipeline.dart         picks label or food mode
    storage.dart               history and profile, stored on the device
  screens/     home (camera), results, history, profile
backend/       FastAPI service that calls the vision model (keeps your API key off the phone)
test/          parser and analyzer unit tests (flutter test)
```

## Make it yours

- **Bigger knowledge base:** `lib/data/knowledge.dart` is a starter set. Replace or extend it with
  a full additives dataset, or look ingredients up in Open Food Facts.
- **Barcode scanning:** add a barcode mode and fetch the product from Open Food Facts. It is a strong
  fallback when a label is curved, blurry or in another language.
- **More languages:** OCR already reads Latin scripts. Add keyword lists per language in `knowledge.dart`.
- **Before you publish:** put the backend behind https, add rate limits, and use real per-user auth or
  app attestation. An `APP_KEY` compiled into the app only stops casual misuse.

## Limits worth being upfront about

Text is read from photos and can be misread. Diet and allergen checks are keyword based; they cannot see
certifications or cross-contamination. Photo analysis can only guess at hidden ingredients. The app says
this on every results screen. Do not present it as medical advice.
