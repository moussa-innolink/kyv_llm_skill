## Repositories
- **SDK**: https://github.com/moussa-innolink/kyv_swift
- **Example App**: https://github.com/moussa-innolink/kyb_example_swift

# KyvShield iOS SDK

**KyvShield** — iOS SDK for identity verification (KYC). Selfie liveness, document OCR, face matching.

## Installation (Swift Package Manager)

Add the package dependency in Xcode:

1. **File > Add Package Dependencies...**
2. Enter the repository URL
3. Select the `KyvshieldLite` library

Or add it to your `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/moussa-innolink/kyv_swift.git", from: "0.0.4"),
]
```

Minimum deployment target: **iOS 15.0**

## Full Example

All possible enum values listed. Code is copy-paste ready.

```swift
import KyvshieldLite

// ── Possible values ───────────────────────────────────────────────────
// CaptureStep        : .selfie, .recto, .verso
// ChallengeMode      : .minimal, .standard, .strict
// SelfieDisplayMode  : .standard, .compact, .immersive, .neonHud
// DocumentDisplayMode: .standard, .compact, .immersive, .neonHud
// ChallengeAudioRepeat: .once, .twice, .thrice

// ── Fetch documents from API ──────────────────────────────────────────

let url = URL(string: "https://kyvshield-naruto.innolinkcloud.com/api/v1/documents")!
var request = URLRequest(url: url)
request.setValue("YOUR_API_KEY", forHTTPHeaderField: "X-API-Key")
let (data, _) = try await URLSession.shared.data(for: request)
let json = try JSONSerialization.jsonObject(with: data) as! [String: Any]
let docsJson = json["documents"] as! [[String: Any]]
let docs = docsJson.map { KyvshieldDocument.fromJson($0) }
let selectedDoc = docs.first!

// ── Request camera permission ─────────────────────────────────────────

KyvshieldLite.requestCameraPermission { granted in
    guard granted else {
        if KyvshieldLite.isCameraPermissionDenied() {
            KyvshieldLite.openSettings()
        }
        return
    }
    startKyc(doc: selectedDoc)
}

// ── Start KYC (UIKit) ─────────────────────────────────────────────────

func startKyc(doc: KyvshieldDocument) {
    let config = KyvshieldConfig(
        baseUrl: "https://kyvshield-naruto.innolinkcloud.com",
        apiKey: "YOUR_API_KEY",
        enableLog: true,
        theme: KyvshieldThemeConfig(
            primaryColor: UIColor(hex: "#EF8352"),
            brightness: .light
        )
    )

    let flow = KyvshieldFlowConfig(
        steps: [.selfie, .recto, .verso],
        target: doc,
        kycIdentifier: "user-12345",
        challengeMode: .minimal,
        stepChallengeModes: [
            .selfie: .minimal,
            .recto: .standard,
            .verso: .minimal,
        ],
        selfieDisplayMode: .standard,
        documentDisplayMode: .standard,
        requireFaceMatch: true,
        requireAml: true,
        showIntroPage: true,
        showInstructionPages: true,
        showResultPage: true,
        showSuccessPerStep: true,
        language: "fr",
        playChallengeAudio: true,
        maxChallengeAudioPlay: .once,
        pauseBetweenAudioPlayMs: 1000
    )

    KyvshieldLite.initKyc(from: self, config: config, flow: flow) { result in
        self.handleResult(result)
    }
}

// ── SwiftUI alternative ───────────────────────────────────────────────

// .fullScreenCover(isPresented: $showKyc) {
//     KyvshieldView(config: config, flow: flow) { result in
//         showKyc = false
//         handleResult(result)
//     }
// }

// ── Handle result ─────────────────────────────────────────────────────

func handleResult(_ result: KYCResult) {
    print("Success: \(result.success)")
    print("Status: \(result.overallStatus)")   // .pass, .reject, .error
    print("Session: \(result.sessionId ?? "")")

    // Selfie
    if let selfie = result.selfieResult {
        print("Live: \(selfie.isLive)")
        print("Image: \(selfie.capturedImage?.count ?? 0) bytes")
    }

    // Recto
    if let recto = result.rectoResult {
        print("Recto score: \(recto.score)")
        print("Aligned doc: \(recto.alignedDocument?.count ?? 0) bytes")
        print("Photos: \(recto.extractedPhotos.count)")
        print("Fields: \(recto.extraction?.fields.count ?? 0)")
        if let fv = recto.faceVerification {
            print("Face match: \(fv.isMatch)")
            print("Similarity: \(fv.similarityScore)")
        }
    }

    // Verso
    if let verso = result.versoResult {
        print("Verso score: \(verso.score)")
        print("Fields: \(verso.extraction?.fields.count ?? 0)")
    }

    // Extracted data (searches recto + verso)
    print("Nom: \(result.getExtractedValue("nom") ?? "")")
    print("NIN: \(result.getExtractedValue("nin") ?? "")")

    // Loop all fields sorted by priority
    for field in result.rectoResult?.extraction?.sortedFields ?? [] {
        print("\(field.label): \(field.stringValue ?? "")")
    }
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

```swift
KyvshieldLite.checkCameraPermission()                   // Bool
KyvshieldLite.requestCameraPermission { granted in }     // async callback
KyvshieldLite.isCameraPermissionDenied()                 // Bool
KyvshieldLite.openSettings()                             // opens Settings app
```

## Platform Setup (Info.plist)

Add the camera usage description to your `Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera access is required for identity verification.</string>
```

Minimum deployment target: **iOS 15.0**

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

## License

BSD 3-Clause License. See [LICENSE](LICENSE).
