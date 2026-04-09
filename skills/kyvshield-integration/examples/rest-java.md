# KyvShield REST SDK — Java

Server-to-server KYC verification SDK. Wraps the REST API with Builder-pattern options, typed models, and webhook signature verification.

## Installation

### Gradle (JitPack)

```groovy
repositories {
    maven { url 'https://jitpack.io' }
}

dependencies {
    implementation 'com.github.moussa-innolink.kyv_shield_restfull:java:1.2.0'
}
```

### Maven (JitPack)

```xml
<repositories>
    <repository>
        <id>jitpack.io</id>
        <url>https://jitpack.io</url>
    </repository>
</repositories>

<dependency>
    <groupId>com.github.moussa-innolink.kyv_shield_restfull</groupId>
    <artifactId>java</artifactId>
    <version>1.2.0</version>
</dependency>
```

Requires Java 11+.

## Quick Start

```java
import sn.innolink.kyvshield.KyvShield;

KyvShield client = new KyvShield("YOUR_API_KEY", "https://kyvshield-naruto.innolinkcloud.com");
```

The second argument (base URL) is optional and defaults to the production endpoint.

## Get Available Challenges

```java
import sn.innolink.kyvshield.model.Challenges;

Challenges challenges = client.getChallenges();
// challenges.getDocument().getMinimal()  → ["center_document"]
// challenges.getDocument().getStandard() → ["center_document", "tilt_left", "tilt_right"]
// challenges.getSelfie().getMinimal()    → ["center_face", "close_eyes"]
```

## Verify — Full KYC

Image keys follow the `{step}_{challenge}` convention. Use the Builder to construct options.

```java
import sn.innolink.kyvshield.model.*;

KycResponse result = client.verify(
    new VerifyOptions.Builder()
        .steps(Step.SELFIE, Step.RECTO, Step.VERSO)
        .target("SN-CIN")
        .language("fr")
        .challengeMode(ChallengeMode.MINIMAL)
        .selfieChhallengeMode(ChallengeMode.MINIMAL)
        .rectoChallengeMode(ChallengeMode.MINIMAL)
        .versoChallengeMode(ChallengeMode.MINIMAL)
        .requireFaceMatch(true)
        .kycIdentifier("user-12345")
        .addImage("selfie_center_face", "./selfie_centered.jpg")
        .addImage("selfie_close_eyes", "./selfie_eyes_closed.jpg")
        .addImage("recto_center_document", "./cin_recto.jpg")
        .addImage("verso_center_document", "./cin_verso.jpg")
        .build()
);

System.out.println(result.getOverallStatus());     // "pass" | "reject"
System.out.println(result.getSessionId());
System.out.println(result.getOverallConfidence());
```

### Recto Only (minimal)

```java
KycResponse result = client.verify(
    new VerifyOptions.Builder()
        .steps(Step.RECTO)
        .target("SN-CIN")
        .language("fr")
        .rectoChallengeMode(ChallengeMode.MINIMAL)
        .addImage("recto_center_document", "./cin_recto.jpg")
        .build()
);
```

### Using byte[] Instead of File Paths

```java
byte[] rectoBytes = Files.readAllBytes(Path.of("cin_recto.jpg"));

KycResponse result = client.verify(
    new VerifyOptions.Builder()
        .steps(Step.RECTO)
        .target("SN-CIN")
        .language("fr")
        .rectoChallengeMode(ChallengeMode.MINIMAL)
        .addImageBytes("recto_center_document", rectoBytes)
        .build()
);
```

## Image Input Formats

The `addImage` method accepts multiple formats. The SDK detects the type automatically.

| Method | Input | Description |
|--------|-------|-------------|
| `addImage(key, path)` | `"./recto.jpg"` | Local file path |
| `addImage(key, url)` | `"https://example.com/recto.jpg"` | URL, downloaded before upload |
| `addImage(key, dataUri)` | `"data:image/jpeg;base64,/9j/..."` | Data URI |
| `addImage(key, base64)` | `"/9j/4AAQSkZJRgABA..."` | Raw base64 string |
| `addImageBytes(key, bytes)` | `byte[]` | In-memory byte array |

## Batch Verification

Submit up to 10 verifications concurrently. Returns a list of results in the same order.

```java
import java.util.List;

List<KycResponse> results = client.verifyBatch(List.of(
    new VerifyOptions.Builder()
        .steps(Step.RECTO)
        .target("SN-CIN")
        .language("fr")
        .rectoChallengeMode(ChallengeMode.MINIMAL)
        .addImage("recto_center_document", "./user1_recto.jpg")
        .build(),
    new VerifyOptions.Builder()
        .steps(Step.RECTO, Step.VERSO)
        .target("SN-CIN")
        .language("fr")
        .rectoChallengeMode(ChallengeMode.MINIMAL)
        .versoChallengeMode(ChallengeMode.MINIMAL)
        .addImage("recto_center_document", "./user2_recto.jpg")
        .addImage("verso_center_document", "./user2_verso.jpg")
        .build()
));

for (int i = 0; i < results.size(); i++) {
    System.out.printf("Verification %d: %s%n", i, results.get(i).getOverallStatus());
}
```

## Webhook Signature Verification

Static method — does not require a client instance.

```java
import sn.innolink.kyvshield.KyvShield;

// In your controller (e.g., Spring Boot)
@PostMapping("/webhook")
public ResponseEntity<String> webhook(
        @RequestBody byte[] rawBody,
        @RequestHeader("X-Kyvshield-Signature") String signature,
        @RequestHeader("X-Kyvshield-Event") String event,
        @RequestHeader("X-Kyvshield-Session") String sessionId) {

    boolean valid = KyvShield.verifyWebhookSignature(rawBody, "YOUR_API_KEY", signature);
    if (!valid) {
        return ResponseEntity.status(401).body("Invalid signature");
    }

    // Process event...
    return ResponseEntity.ok("OK");
}
```

## Response Structure

```java
public class KycResponse {
    boolean success;
    String sessionId;
    String overallStatus;           // "pass" | "reject"
    double overallConfidence;
    FaceVerification faceVerification;  // nullable
    List<StepResult> steps;
    int processingTimeMs;
}

public class StepResult {
    int stepIndex;
    String stepType;                // "selfie" | "recto" | "verso"
    boolean success;
    Liveness liveness;
    Verification verification;      // nullable
    List<ExtractionField> extraction;  // nullable
    List<ExtractedPhoto> extractedPhotos;  // nullable
    String alignedDocument;         // base64 JPEG (recto/verso)
    String capturedImage;           // base64 JPEG (selfie)
    int processingTimeMs;
}

public class ExtractionField {
    String key;
    String documentKey;
    String label;
    String value;
    int displayPriority;
}
```

## Error Handling

```java
import sn.innolink.kyvshield.exception.*;

try {
    KycResponse result = client.verify(options);
} catch (AuthenticationException e) {
    // 401 — Invalid API key
    System.err.println("Auth error: " + e.getMessage());
} catch (ValidationException e) {
    // 422 — Missing or invalid fields
    System.err.println("Validation: " + e.getMessage());
    System.err.println("Details: " + e.getDetails());
} catch (KyvShieldException e) {
    // Other API errors (400, 500, timeout)
    System.err.printf("Error [%d]: %s%n", e.getStatusCode(), e.getMessage());
}
```

## Notes

- **Timeout**: 120 seconds (SDK default, configurable)
- **Max body size**: 30 MB total across all images
- **Image format**: JPEG recommended
- **Processing time**: 10-60 seconds depending on steps and challenge modes
- **Thread safety**: `KyvShield` client instance is thread-safe

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

Source: [github.com/moussa-innolink/kyv_shield_restfull](https://github.com/moussa-innolink/kyv_shield_restfull)
