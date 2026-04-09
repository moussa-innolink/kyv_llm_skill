# KyvShield REST SDK — Kotlin

Server-to-server KYC verification SDK. Wraps the REST API with idiomatic Kotlin data classes, coroutine support, and webhook signature verification.

## Installation

### Gradle (JitPack)

```kotlin
repositories {
    maven("https://jitpack.io")
}

dependencies {
    implementation("com.github.moussa-innolink.kyv_shield_restfull:kotlin:1.2.0")
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
    <artifactId>kotlin</artifactId>
    <version>1.2.0</version>
</dependency>
```

Requires Kotlin 1.8+ / Java 11+.

## Quick Start

```kotlin
import sn.innolink.kyvshield.KyvShield

val client = KyvShield("YOUR_API_KEY", "https://kyvshield-naruto.innolinkcloud.com")
```

The second argument (base URL) is optional and defaults to the production endpoint.

## Get Available Challenges

```kotlin
val challenges = client.getChallenges()
// challenges.document.minimal  → ["center_document"]
// challenges.document.standard → ["center_document", "tilt_left", "tilt_right"]
// challenges.selfie.minimal    → ["center_face", "close_eyes"]
```

## Verify — Full KYC

Image keys follow the `{step}_{challenge}` convention.

```kotlin
import sn.innolink.kyvshield.model.*

val result = client.verify(VerifyOptions(
    steps = listOf(Step.SELFIE, Step.RECTO, Step.VERSO),
    target = "SN-CIN",
    language = Language.FRENCH,
    challengeMode = ChallengeMode.MINIMAL,
    selfieChhallengeMode = ChallengeMode.MINIMAL,
    rectoChallengeMode = ChallengeMode.MINIMAL,
    versoChallengeMode = ChallengeMode.MINIMAL,
    requireFaceMatch = true,
    kycIdentifier = "user-12345",
    images = mapOf(
        "selfie_center_face" to "./selfie_centered.jpg",
        "selfie_close_eyes" to "./selfie_eyes_closed.jpg",
        "recto_center_document" to "./cin_recto.jpg",
        "verso_center_document" to "./cin_verso.jpg",
    ),
))

println(result.overallStatus)      // "pass" | "reject"
println(result.sessionId)
println(result.overallConfidence)
```

### Recto Only (minimal)

```kotlin
val result = client.verify(VerifyOptions(
    steps = listOf(Step.RECTO),
    target = "SN-CIN",
    language = Language.FRENCH,
    rectoChallengeMode = ChallengeMode.MINIMAL,
    images = mapOf(
        "recto_center_document" to "./cin_recto.jpg",
    ),
))
```

### Using ByteArray Instead of File Paths

```kotlin
val rectoBytes: ByteArray = File("cin_recto.jpg").readBytes()

val result = client.verify(VerifyOptions(
    steps = listOf(Step.RECTO),
    target = "SN-CIN",
    language = Language.FRENCH,
    rectoChallengeMode = ChallengeMode.MINIMAL,
    images = mapOf(
        "recto_center_document" to "./cin_recto.jpg",
    ),
    imageBytes = mapOf(
        "verso_center_document" to versoBytes,
    ),
))
```

## Image Input Formats

The `images` map values accept multiple formats. The SDK detects the type automatically.

| Format | Example | Description |
|--------|---------|-------------|
| File path | `"./recto.jpg"` | Local file, read from disk |
| URL | `"https://example.com/recto.jpg"` | Downloaded by SDK before upload |
| Data URI | `"data:image/jpeg;base64,/9j/..."` | Standard data URI |
| Base64 string | `"/9j/4AAQSkZJRgABA..."` | Raw base64 (no prefix) |

The `imageBytes` map accepts `ByteArray` values for in-memory image data.

## Batch Verification

Submit up to 10 verifications concurrently. Returns a list of results in the same order.

```kotlin
val results = client.verifyBatch(listOf(
    VerifyOptions(
        steps = listOf(Step.RECTO),
        target = "SN-CIN",
        language = Language.FRENCH,
        rectoChallengeMode = ChallengeMode.MINIMAL,
        images = mapOf("recto_center_document" to "./user1_recto.jpg"),
    ),
    VerifyOptions(
        steps = listOf(Step.RECTO, Step.VERSO),
        target = "SN-CIN",
        language = Language.FRENCH,
        rectoChallengeMode = ChallengeMode.MINIMAL,
        versoChallengeMode = ChallengeMode.MINIMAL,
        images = mapOf(
            "recto_center_document" to "./user2_recto.jpg",
            "verso_center_document" to "./user2_verso.jpg",
        ),
    ),
))

results.forEachIndexed { i, r ->
    println("Verification $i: ${r.overallStatus}")
}
```

## Webhook Signature Verification

Companion object method — does not require a client instance.

```kotlin
import sn.innolink.kyvshield.KyvShield

// In your controller (e.g., Ktor)
post("/webhook") {
    val rawBody = call.receiveText().toByteArray()
    val signature = call.request.headers["X-Kyvshield-Signature"] ?: ""

    val valid = KyvShield.verifyWebhookSignature(rawBody, "YOUR_API_KEY", signature)
    if (!valid) {
        call.respond(HttpStatusCode.Unauthorized, "Invalid signature")
        return@post
    }

    val event = call.request.headers["X-Kyvshield-Event"]
    val sessionId = call.request.headers["X-Kyvshield-Session"]
    // Process event...
    call.respond(HttpStatusCode.OK, "OK")
}
```

### Spring Boot Example

```kotlin
@PostMapping("/webhook")
fun webhook(
    @RequestBody rawBody: ByteArray,
    @RequestHeader("X-Kyvshield-Signature") signature: String,
    @RequestHeader("X-Kyvshield-Event") event: String,
    @RequestHeader("X-Kyvshield-Session") sessionId: String,
): ResponseEntity<String> {
    val valid = KyvShield.verifyWebhookSignature(rawBody, "YOUR_API_KEY", signature)
    if (!valid) return ResponseEntity.status(401).body("Invalid signature")

    // Process event...
    return ResponseEntity.ok("OK")
}
```

## Response Structure

```kotlin
data class KycResponse(
    val success: Boolean,
    val sessionId: String,
    val overallStatus: String,           // "pass" | "reject"
    val overallConfidence: Double,
    val faceVerification: FaceVerification?,
    val steps: List<StepResult>,
    val processingTimeMs: Int,
)

data class StepResult(
    val stepIndex: Int,
    val stepType: String,                // "selfie" | "recto" | "verso"
    val success: Boolean,
    val liveness: Liveness,
    val verification: Verification?,
    val extraction: List<ExtractionField>?,
    val extractedPhotos: List<ExtractedPhoto>?,
    val alignedDocument: String?,        // base64 JPEG (recto/verso)
    val capturedImage: String?,          // base64 JPEG (selfie)
    val processingTimeMs: Int,
)

data class ExtractionField(
    val key: String,
    val documentKey: String,
    val label: String,
    val value: String,
    val displayPriority: Int,
)
```

## Error Handling

```kotlin
import sn.innolink.kyvshield.exception.*

try {
    val result = client.verify(options)
} catch (e: AuthenticationException) {
    // 401 — Invalid API key
    println("Auth error: ${e.message}")
} catch (e: ValidationException) {
    // 422 — Missing or invalid fields
    println("Validation: ${e.message}")
    println("Details: ${e.details}")
} catch (e: KyvShieldException) {
    // Other API errors (400, 500, timeout)
    println("Error [${e.statusCode}]: ${e.message}")
}
```

## Notes

- **Timeout**: 120 seconds (SDK default, configurable)
- **Max body size**: 30 MB total across all images
- **Image format**: JPEG recommended
- **Processing time**: 10-60 seconds depending on steps and challenge modes
- **Coroutines**: All methods are `suspend` functions for Kotlin coroutine support
- **Thread safety**: `KyvShield` client instance is thread-safe

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

Source: [github.com/moussa-innolink/kyv_shield_restfull](https://github.com/moussa-innolink/kyv_shield_restfull)
