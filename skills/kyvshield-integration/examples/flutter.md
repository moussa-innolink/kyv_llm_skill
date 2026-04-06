## Repositories
- **SDK**: https://github.com/moussa-innolink/kyv_flutter_lite
- **Example App**: https://github.com/moussa-innolink/kyb_example_flutter

# KyvShield Flutter SDK

**KyvShield** — Flutter SDK for identity verification (KYC). Selfie liveness, document OCR, face matching.

[![pub package](https://img.shields.io/pub/v/kyvshield_lite.svg)](https://pub.dev/packages/kyvshield_lite)

## Installation

```yaml
dependencies:
  kyvshield_lite: ^0.0.3
```

## Full Example

All possible enum values listed. Code is copy-paste ready.

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'package:kyvshield_lite/kyvshield_lite.dart';

// ── Possible values ───────────────────────────────────────────────────
// CaptureStep        : selfie, recto, verso
// ChallengeMode      : minimal, standard, strict
// SelfieDisplayMode  : standard, compact, immersive, neonHud
// DocumentDisplayMode: standard, compact, immersive, neonHud
// ChallengeAudioRepeat: once, twice, thrice
// Brightness         : light, dark

// ── Fetch available documents from API ────────────────────────────────

final res = await http.get(
  Uri.parse('https://kyvshield-naruto.innolinkcloud.com/api/v1/documents'),
  headers: {'X-API-Key': 'YOUR_API_KEY'},
);
final docs = (jsonDecode(res.body)['documents'] as List)
    .map((d) => KyvshieldDocument.fromJson(d))
    .toList();
final selectedDoc = docs.first; // or let user pick

// ── Request camera permission ─────────────────────────────────────────

final granted = await Kyvshield.requestCameraPermission();
if (!granted) {
  final denied = await Kyvshield.isCameraPermissionPermanentlyDenied();
  if (denied) await Kyvshield.openSettings();
  return;
}

// ── Start KYC ─────────────────────────────────────────────────────────

final result = await Kyvshield.initKyc(
  context: context,
  config: KyvshieldConfig(
    baseUrl: 'https://kyvshield-naruto.innolinkcloud.com',
    apiKey: 'YOUR_API_KEY',
    enableLog: true,
    theme: KyvshieldThemeConfig(
      primaryColor: Color(0xFFEF8352),
      brightness: Brightness.light,
    ),
  ),
  flow: KyvshieldFlowConfig(
    steps: [CaptureStep.selfie, CaptureStep.recto, CaptureStep.verso],
    language: 'fr',
    showIntroPage: true,
    showInstructionPages: true,
    showResultPage: true,
    showSuccessPerStep: true,
    selfieDisplayMode: SelfieDisplayMode.standard,
    documentDisplayMode: DocumentDisplayMode.standard,
    challengeMode: ChallengeMode.minimal,
    requireFaceMatch: true,
    playChallengeAudio: true,
    maxChallengeAudioPlay: ChallengeAudioRepeat.once,
    pauseBetweenAudioPlay: Duration(seconds: 1),
    stepChallengeModes: {
      CaptureStep.selfie: ChallengeMode.minimal,
      CaptureStep.recto: ChallengeMode.standard,
      CaptureStep.verso: ChallengeMode.minimal,
    },
    target: selectedDoc,
    kycIdentifier: 'user-12345',
  ),
);

// ── Handle result ─────────────────────────────────────────────────────

print('Success: ${result.success}');
print('Status: ${result.overallStatus}');   // pass, reject, error
print('Session: ${result.sessionId}');

// Selfie
if (result.selfieResult != null) {
  print('Live: ${result.selfieResult!.isLive}');
  print('Image: ${result.selfieResult!.capturedImage?.length} bytes');
}

// Recto
if (result.rectoResult != null) {
  print('Recto score: ${result.rectoResult!.score}');
  print('Aligned doc: ${result.rectoResult!.alignedDocument?.length} bytes');
  print('Photos: ${result.rectoResult!.extractedPhotos.length}');
  print('Fields: ${result.rectoResult!.extraction?.fields.length}');
  // Face match (attached to recto)
  if (result.rectoResult!.faceVerification != null) {
    print('Face match: ${result.rectoResult!.faceVerification!.isMatch}');
    print('Similarity: ${result.rectoResult!.faceVerification!.similarityScore}');
  }
}

// Verso
if (result.versoResult != null) {
  print('Verso score: ${result.versoResult!.score}');
  print('Fields: ${result.versoResult!.extraction?.fields.length}');
}

// Extracted data (searches recto + verso)
print('Nom: ${result.getExtractedValue("nom")}');
print('NIN: ${result.getExtractedValue("nin")}');

// Loop all fields sorted by priority
for (final field in result.rectoResult?.extraction?.sortedFields ?? []) {
  print('${field.label}: ${field.stringValue}');
}
```

## Display Modes

| Mode | Description |
|------|-------------|
| `standard` | Classic layout with header, camera, and instructions below |
| `compact` | Camera fills screen, instructions overlay at bottom |
| `immersive` | Full-screen camera with glass-effect overlays |
| `neonHud` | Futuristic dark theme with glow effects and monospace font |

## Permission Helpers

```dart
await Kyvshield.checkCameraPermission();
await Kyvshield.requestCameraPermission();
await Kyvshield.isCameraPermissionPermanentlyDenied();
await Kyvshield.openSettings();
```

## Platform Setup

### Android

`android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
```

Min SDK: **21** (Android 5.0 Lollipop)

### iOS

`ios/Runner/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera access is required for identity verification.</string>
```

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

## License

BSD 3-Clause License. See [LICENSE](LICENSE).
