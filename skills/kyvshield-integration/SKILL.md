---
name: kyvshield
description: Integrate KyvShield KYC SDK into any app. Use when building identity verification, integrating KYC, adding selfie liveness, document scanning, face matching, or handling KyvShield webhooks. Supports Web, Flutter, Android/Kotlin, iOS/Swift, and React Native.
argument-hint: [platform] [task]
---

# KyvShield SDK Integration Guide

You are an expert at integrating the KyvShield KYC SDK. KyvShield provides identity verification with selfie liveness detection, document OCR, and face matching.

## Platforms

KyvShield is available on 5 platforms. All share the same API surface and result types.

| Platform | Package | Install |
|---|---|---|
| **Web** | CDN | `<script src="https://kyvshield-naruto.innolinkcloud.com/static/sdk/kyvshield.min.js">` |
| **Flutter** | Git | `kyvshield_lite` from `github.com/moussa-innolink/kyv_flutter_lite` |
| **Android** | JitPack | `com.github.moussa-innolink:kyv_android:0.0.4` |
| **iOS** | SPM | `github.com/moussa-innolink/kyv_swift` version `0.0.4` |
| **React Native** | npm | `@kyvshield/react-native-lite@0.0.4` |

## API URLs

| Environment | URL |
|---|---|
| Development | `https://kyvshield.innolink.sn` |
| Production | `https://kyvshield-naruto.innolinkcloud.com` |

## Integration Steps (all platforms)

1. **Fetch available documents** from `GET /api/v1/documents` with header `X-API-Key`
2. **Request camera permission**
3. **Configure** `KyvshieldConfig` with `baseUrl` and `apiKey`
4. **Configure flow** `KyvshieldFlowConfig` with steps, language, display mode, etc.
5. **Launch KYC** via `initKyc(config, flow)`
6. **Handle result** `KYCResult` with success, overallStatus, selfieResult, rectoResult, versoResult

For platform-specific code examples, see the supporting files:
- [Web integration](examples/web.md)
- [Flutter integration](examples/flutter.md)
- [Android/Kotlin integration](examples/android.md)
- [iOS/Swift integration](examples/swift.md)
- [React Native integration](examples/reactnative.md)
- [Webhook handling](examples/webhooks.md)

## Configuration Reference

### KyvshieldConfig
```
baseUrl         String (required)  - API server URL
apiKey          String (required)  - API key
apiVersion      String = "v1"      - API version
enableLog       bool = false       - Debug logs
timeoutSeconds  int = 60           - HTTP timeout
theme           ThemeConfig?       - Colors + dark mode
```

### KyvshieldFlowConfig
```
steps                  CaptureStep[] = [selfie, recto, verso]
language               String = "fr"     - "fr" | "en" | "wo"
challengeMode          ChallengeMode = standard  - minimal | standard | strict
selfieDisplayMode      DisplayMode = standard    - standard | compact | immersive | neonHud
documentDisplayMode    DisplayMode = standard
showIntroPage          bool = true
showInstructionPages   bool = true
showResultPage         bool = true
showSuccessPerStep     bool = true
requireFaceMatch       bool = true
playChallengeAudio     bool = false
target                 KyvshieldDocument?  - from /api/v1/documents
kycIdentifier          String?  - your reference ID (returned in webhooks)
```

## Result Types

### KYCResult
```
success              bool           - KYC completed without error
overallStatus        String         - "pass" | "reject" | "error"
sessionId            String?        - server session ID
selfieResult         SelfieResult?  - liveness result
rectoResult          DocumentResult? - front side OCR + fraud
versoResult          DocumentResult? - back side (NIN, MRZ)
totalProcessingTimeMs int
errorMessage         String?

Convenience:
  getExtractedValue(key)  -> String?   - search recto + verso fields
  selfieImage             -> bytes?    - captured selfie
  rectoImage              -> bytes?    - aligned recto document
  versoImage              -> bytes?    - aligned verso document
  faceMatches             -> bool?     - selfie matches document
  faceSimilarityScore     -> double?   - match score 0-100%
  mainPhoto               -> ExtractedPhoto? - face from document
  authenticityScore       -> double    - average document score
  allExtractedPhotos      -> ExtractedPhoto[]
```

### SelfieResult
```
success           bool
isLive            bool       - real person (not photo/screen)
confidence        double     - 0.0 to 1.0
status            String     - "pass" | "reject"
capturedImage     bytes?     - JPEG selfie
challengesPassed  int
challengesTotal   int
processingTimeMs  int
```

### DocumentResult
```
success           bool
isLive            bool       - real document (not screen)
score             double     - authenticity 0.0 to 1.0
confidenceLevel   String     - "HIGH" | "MEDIUM" | "LOW"
status            String     - "pass" | "reject"
alignedDocument   bytes?     - perspective-corrected JPEG
extraction        DocumentData? - OCR fields
extractedPhotos   ExtractedPhoto[] - face photos from document
faceVerification  FaceResult? - selfie vs document match
processingTimeMs  int
```

### DocumentData
```
fields                  ExtractedField[]
documentType            String?
sortedFields            -> fields sorted by displayPriority
getField(key)           -> ExtractedField?
getValue(key)           -> String?
```

### ExtractedField
```
key              String  - generic key (document_id, first_name...)
documentKey      String  - document-specific key (numero_carte, prenoms...)
label            String  - localized display label
value            any     - extracted value
displayPriority  int     - sort order (1 = document identifier, lower = show first)
icon             String? - icon name (user, calendar, credit_card...)
```

**displayPriority convention:**
- `1` = always the document's main identifier (N° carte for recto, NIN for verso)
- `2-10` = important fields (name, birth date, etc.)
- `99+` = technical fields (MRZ, codes)

### FaceResult
```
isMatch           bool     - faces match
similarityScore   double   - 0-100%
threshold         double   - match threshold
confidenceLevel   String   - VERY_HIGH | HIGH | MEDIUM | LOW
detectionModel    String   - e.g. "scrfd_10g"
recognitionModel  String   - e.g. "buffalo_l"
processingTimeMs  int
```

### ExtractedPhoto
```
imageBytes   bytes/base64  - face photo JPEG
confidence   double        - detection confidence
width        int
height       int
bbox         double[]      - bounding box [x1, y1, x2, y2]
```

## Webhooks

KyvShield sends webhooks after each step and at session end. See [webhooks.md](examples/webhooks.md) for full examples.

**Events:** `selfie.completed`, `selfie.failed`, `recto.completed`, `recto.failed`, `verso.completed`, `verso.failed`, `session.completed`, `session.failed`

**Headers:**
```
X-KyvShield-Event:     selfie.completed
X-KyvShield-Session:   <session_id>
X-KyvShield-Timestamp: 2026-04-06T22:37:55Z
X-KyvShield-Signature: sha256=<HMAC-SHA256 of body with API key>
Content-Type:          application/json
User-Agent:            KyvShield-Webhook/1.0
```

**Signature verification:** HMAC-SHA256 of the raw body using the API key as secret. Compare with `X-KyvShield-Signature` header (strip `sha256=` prefix).

## Repositories

### SDKs
| Platform | Repository |
|---|---|
| Flutter | https://github.com/moussa-innolink/kyv_flutter_lite |
| Android | https://github.com/moussa-innolink/kyv_android |
| iOS | https://github.com/moussa-innolink/kyv_swift |
| React Native | https://github.com/moussa-innolink/kyv_react_native |

### Example Apps
| Platform | Repository |
|---|---|
| Flutter | https://github.com/moussa-innolink/kyb_example_flutter |
| Android | https://github.com/moussa-innolink/kyb_example_android |
| iOS | https://github.com/moussa-innolink/kyb_example_swift |
| React Native | https://github.com/moussa-innolink/kyb_example_react_native |
| Web | https://github.com/moussa-innolink/kyb_example_web |

### Documentation
- Developer Portal: https://kyvshield-naruto.innolinkcloud.com/developer
